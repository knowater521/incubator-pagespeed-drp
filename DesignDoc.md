# Domain Registry Provider Design Document #

Bryan McQuade

## Objective ##

Reduce binary size and memory footprint of the `RegistryControlledDomainService` implementation without impacting its startup or runtime performance.


## Background ##

Page Speed currently uses a subset of the Chromium codebase. This includes Chromium base, and a few utilities in net/base.

In net/base, we use registry\_controlled\_domain.cc to determine the public suffix for a given hostname. This class provides an API to search through the list of rules in the effective\_tld\_names.dat file provided by Mozilla at http://publicsuffix.org/list/.

Chromium uses a gperf-based implementation to search through the list of rules. The implementation is very runtime-efficient and requires no startup time or heap-allocated storage. However it adds ~170kB to 32-bit binaries, and ~200kB to 64-bit binaries.

Page Speed would like an implementation that reduces binary size and memory footprint but is also startup and runtime efficient. Since Page Speed bundles one binary per target platform in its distribution, the overhead for registry\_controlled\_domain.cc is roughly 910kB (3 32-bit platforms, 2 64-bit platforms).

In addition, browsers (especially mobile browsers) would benefit from a small, efficient registry lookup implementation.


## Proposed implementation ##

### Summary ###

The proposed trie-based implementation has a data footprint of ~30kB, resulting in a binary size reduction of 140kB on 32-bit platforms and 170kB on 64-bit platforms. The trie-based implementation is slightly less runtime efficient, adding ~20% additional runtime. This additional runtime is negligible, however further optimizations are possible if it is necessary to achieve runtime performance equivalent to the current implementation.

### Details ###

I propose a trie-based implementation that splits each rule into its hostname-parts and stores the unique hostname-parts in a string table ([original bug report](http://code.google.com/p/chromium/issues/detail?id=43880)). For instance, the rules "gov.it" and "edu.it" would be split into "gov", "edu", and "it" and stored in the string table as "gov\0edu\0it\0". This implementation can be made to work with no startup cost and ~30kB binary size cost, for a savings of 140kB on 32-bit platforms and 170kB on 64-bit platforms.

The proposed string table can be stored in ~20kB, compared with ~37kB for a string table that contains each rule in full (i.e. "gov.it\0edu.it\0"). Further reductions of about 1500 bytes are possible by eliminating hostname-parts that are suffixes of other hostname-parts (e.g. instead of storing "missoula" and "fortmissoula" as separate entries "missoula\0fortmissoula\0", we store only "fortmissoula\0") for a total savings of ~20kB over storing each rule in full.

With a string table of hostname-parts, we need a mechanism to traverse the hostname-parts to determine the top-level domain for a given hostname. We can do so using a trie. Using the "gov.it" "edu.it" example, we place "it" at the root of the trie, and give it two children "gov" and "edu".

Each trie node can be represented in 5 bytes and stored in an array using the following structure:

```
const char* kStringTable = "it\0edu\0gov";

#pragma pack(push)
#pragma pack(1)

// TrieNode represents a single node in a Trie. It uses 5 bytes of
// storage.
struct TrieNode {
  // Index in the string table for the hostname-part associated with
  // this node.
  unsigned int string_table_offset  : 15;

  // Offset of the first child of this node in the node table. All
  // children are stored adjacent to each other, sorted
  // lexicographically by their hostname parts.
  unsigned int first_child_offset   : 13;

  // Number of children of this node.
  unsigned int num_children         : 11;

  // Whether this node is a "terminal" node. A terminal node is one
  // that represents the end of a sequence of nodes in the trie. For
  // instance if the sequences "com.foo.bar" and "com.foo" are added
  // to the trie, "bar" and "foo" are terminal nodes, since they are
  // both at the end of their sequences.
  unsigned int is_terminal          :  1;
};

#pragma pack(pop)

static const struct TrieNode kNodeTable[] = {
  { 0, 1, 2, 0 },  // it
  { 3, 0, 0, 1 },  // edu.it
  { 7, 0, 0, 1 },  // gov.it
};
```

We use pragma pack to prevent the compiler from padding the structure to 8 bytes.

`string_table_offset` is an offset into the kStringTable for the given entry, `first_child_offset` is the offset of the first child node of the current node in the `kNodeTable`, `num_children` is the number of children for the node, and `is_terminal` indicates whether the given node is a terminal component for a given public suffix rule (e.g. "edu" and "gov" would be terminal nodes). Some non-leaf nodes can be terminal nodes, which is why `is_terminal` is necessary.

Constructing such a trie using the current public suffix ruleset yields ~3800 nodes, or ~19kB of storage, for a total of ~36kB for both the string table and the node array.

Notice that all leaf nodes in the trie have 0 children and are necessarily roots, so some storage is wasted by storing a full `TrieNode` for each. To reduce space we can partition nodes into two sets: nodes where all siblings are leaf nodes, and nodes where not all siblings are leaf nodes. Nodes where all siblings are leaves can be represented in 2 bytes each (a string\_table\_offset per node) while remaining nodes are represented with a 5-byte `TrieNode`. We find that ~2500 of the ~3800 nodes fit into the former category, and can be represented in 2 bytes each, for a total node storage requirement of 2 x 2500 = 5kB and 5 x 1300 = 6.5kB = 11.5kB total, or ~29kB for string table and the node arrays. Example structures are shown below:

```
static const struct TrieNode kNodeTable[] = {
  { 0, 1, 2, 0 },  // it
};

static const uint16* kLeafTable[] = {
  3,  // edu.it
  7,  // gov.it
};
```

A `first_child_offset` greater than or equal to the number of nodes in `kNodeTable` maps to a node in the `kLeafTable`.

Storing child nodes in lexicographical order allows us to perform a binary search at each node to determine if the next hostname part is in its set of children. Most nodes have only a few children, so the binary search should complete in just a few iterations.

## Performance tests ##

### Results Summary ###

An optimized trie implementation shows an ~20% performance cost over the gperf implementation.. Though the percentage difference is significant, this amounts to a mere 0.05 microseconds additional cost per invocation on average. This should be an acceptable runtime performance cost in Chromium (TODO: confirm with Chromium team members).

### Test Methodolody ###

Tests were run on an Ubuntu Lucid laptop using a test set that includes one entry for each of the ~4000 rules in the publicsuffix dat file. Wildcards were replaced with the string 'wildcard' and the '!' at the beginning of each exception rule was removed. Each rule was then prepended with `www.example`. For instance `com` becomes `www.example.com`, `*.ye` becomes `w3.wildcard.ye`, and `!educ.ar` becomes `www.educ.ar`.

Each test was run 10 times, alternating between control and experiment to minimize possible inaccuracies due to changes in system load. Each test ran 10000 iterations of all the test cases in order, for a total of 3696x10000 tests. Test binaries were compiled with -02 and stripped. Tests were timed with the bash builtin 'time' command.