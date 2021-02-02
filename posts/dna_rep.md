```rust
//! How do we find the origin of replication?
pub mod ori {
    use std::{collections::BTreeMap, usize};
    /// Counts and returns the number of times `pattern`
    /// appears in `str`, The runtime of this procedure is
    /// O(M * N) where `M = s.len()` and `N = pattern.len()`
    pub fn pattern_count(s: &str, pattern: &str) -> usize {
        let mut count = 0;
        for window in s.as_bytes().windows(pattern.len()) {
            if pattern_matches(pattern, window) {
                count += 1;
            }
        }
        count
    }

    /// A linear time procedure to check if `pattern` and `window`
    /// have the same characters in the same order.
    fn pattern_matches(pattern: &str, window: &[u8]) -> bool {
        for (a, b) in pattern.as_bytes().iter().zip(window) {
            if a != b {
                return false;
            }
        }
        true
    }

    /// Returns a count of each k-mer in the given string. This procedure calculates
    /// all counts in linear time
    pub fn count_of_kmers_in(s: &str, k: usize) -> BTreeMap<&str, usize> {
        let mut counter = BTreeMap::new();
        for cur_kmer in s.as_bytes().windows(k) {
            let as_str = std::str::from_utf8(cur_kmer).unwrap();
            let cur_count = counter.entry(as_str).or_insert(0);
            *cur_count += 1;
        }
        counter
    }

    /// Given a map of counts, returns a list of the most frequent ones
    pub fn find_most_frequent_kmers<'a, 'b>(
        kmers_counter: &'a BTreeMap<&'b str, usize>,
    ) -> Vec<(&'a &'b str, &'a usize)> {
        let &max = kmers_counter.values().max().unwrap();
        kmers_counter.iter().filter(|(_, &v)| v == max).collect()
    }

    /// Returns the complementary strand of the given
    /// `dna_strand`. The returned sequence is ordered
    /// 5' -> 3'
    pub fn reverse_complement(dna_strand: &str, complement_mapper: impl Fn(char) -> char) -> String {
        dna_strand
            .chars()
            .map(|c| complement_mapper(c))
            .rev()
            .collect::<String>()
    }

    pub fn deoxyribose_complement(c: char) -> char {
        match c {
            'A' => 'T',
            'a' => 't',
            'T' => 'A',
            't' => 'a',
            'G' => 'C',
            'g' => 'c',
            'C' => 'G',
            'c' => 'g',
            _ => panic!("{} is an invalid dna nucleotide character", c),
        }
    }

    /// Finds all index locations where `patten` occurs in `s`. This is
    /// an O(|s| lg |s|) procedure.
    pub fn find_all_occurances(s: &str, pattern: &str) -> Vec<usize> {
        let mut locations = Vec::new();
        // 1. Create the suffix array by sorting. -- n lg n
        let mut sa = Vec::with_capacity(s.len());
        for i in 0..s.len() {
            sa.push((i, &s[i..]))
        }
        sa.sort_by_key(|(_, suffix)| *suffix);

        // 2. Do a binary search to find an index within the matching region -- ln g
        let pl = pattern.len();
        let search_res = sa.binary_search_by(|(_, suffix)| pattern.cmp(&suffix[..pl]));
        match search_res {
            Err(_) => locations,
            Ok(idx_in_bucket) => {
                // 3. Do a two linear scans: one to the left, another to the right.
                //    This is necessary because we can have multiple matches
                //    and binary search will return any of them, not the first one
                locations.push(sa[idx_in_bucket].0);
                // TODO
                locations
            }
        }
    }

    /// Finds all the distinct k-mers that form (interval_size, t) clumps in genome.
    /// That is, it finds all k-mers that are repeated more than `t` times in any
    /// interval of `interval_size` in `genome`.
    pub fn find_clumping_kmers(genome: &str, interval_size: usize, t: usize, k: usize) -> Vec<&str> {
        todo!()
    }
}

#[cfg(test)]
mod test_ori {
    use super::ori;
    const VC_ORI: &str = "atcaatgatcaacgtaagcttctaagcatgatcaaggtgctcacacagtttatccacaac
    ctgagtggatgacatcaagataggtcgttgtatctccttcctctcgtactctcatgacca
    cggaaagatgatcaagagaggatgatttcttggccatatcgcaatgaatacttgtgactt
    gtgcttccaattgacatcttcagcgccatattgcgctggccaaggtgacggagcgggatt
    acgaaagcatgatcatggctgttgttctgtttatcttgttttgactgagacttgttagga
    tagacggtttttcatcactgactagccaaagccttactctgcctgacatcgaccgtaaat
    tgataatgaatttacatgcttccgcgacgatttacctcttgatcatcgatccgattgaag
    atcttcaattgttaattctcttgcctcgactcatagccatgatgagctcttgatcatgtt
    tccttaaccctctattttttacggaagaatgatcaagctgctgctcttgatcatcgtttc";
    #[test]
    fn pattern_count() {
        let s = "ATCATGATTTTGGCTACTGTAGCTGAT";
        let pattern = "TT";
        let cnt = ori::pattern_count(s, pattern);
        println!("{}", cnt)
    }

    #[test]
    fn find_most_frequent_kmers() {
        // let vc_genome = std::fs::read_to_string("data/vibrio_cholerae.txt");
        // let genome = match vc_genome {
        //     Err(err) => panic!(err),
        //     Ok(genome) => genome,
        // };
        let counter = ori::count_of_kmers_in(VC_ORI, 9);
        let most_frequent_nine_mers = ori::find_most_frequent_kmers(&counter);
        assert_eq!(most_frequent_nine_mers.len(), 4);
        for (k, &v) in most_frequent_nine_mers {
            assert_eq!(v, 3);
            println!("{} appears {} times", k, v);
        }
    }

    #[test]
    fn reverse_complement() {
        let base_strand = "ATGATCAAG";
        let complement = "CTTGATCAT";
        assert_eq!(
            complement,
            ori::reverse_complement(base_strand, ori::deoxyribose_complement)
        );
    }
}
```