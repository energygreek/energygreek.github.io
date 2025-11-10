---
date: 
 created: 2021-02-05
 updated: 2025-07-15
tags: [算法,查找]
---

# 子串查找算法

leetcode的[题目](https://leetcode-cn.com/problems/implement-strstr/), 实现strstr,即查找子字符串

## 最简单的算法

```c
int strStr(char * haystack, char * needle){
    char *tmp = haystack;
    while(*tmp != 0){
        if(strncmp(tmp, needle,strlen(needle))==0){
            return tmp - haystack;
        }
        tmp ++;
    }
    return -1;
}

```

## kmp 算法

```c

// C++ program for implementation of KMP pattern searching 
// algorithm 
#include <bits/stdc++.h> 
  
void computeLPSArray(char* pat, int M, int* lps); 
  
// Prints occurrences of txt[] in pat[] 
void KMPSearch(char* pat, char* txt) 
{ 
    int M = strlen(pat); 
    int N = strlen(txt); 
  
    // create lps[] that will hold the longest prefix suffix 
    // values for pattern 
    int lps[M]; 
  
    // Preprocess the pattern (calculate lps[] array) 
    computeLPSArray(pat, M, lps); 
  
    int i = 0; // index for txt[] 
    int j = 0; // index for pat[] 
    while (i < N) { 
        if (pat[j] == txt[i]) { 
            j++; 
            i++; 
        } 
  
        if (j == M) { 
            printf("Found pattern at index %d ", i - j); 
            j = lps[j - 1]; 
        } 
  
        // mismatch after j matches 
        else if (i < N && pat[j] != txt[i]) { 
            // Do not match lps[0..lps[j-1]] characters, 
            // they will match anyway 
            if (j != 0) 
                j = lps[j - 1]; 
            else
                i = i + 1; 
        } 
    } 
} 
  
// Fills lps[] for given patttern pat[0..M-1] 
void computeLPSArray(char* pat, int M, int* lps) 
{ 
    // length of the previous longest prefix suffix 
    int len = 0; 
  
    lps[0] = 0; // lps[0] is always 0 
  
    // the loop calculates lps[i] for i = 1 to M-1 
    int i = 1; 
    while (i < M) { 
        if (pat[i] == pat[len]) { 
            len++; 
            lps[i] = len; 
            i++; 
        } 
        else // (pat[i] != pat[len]) 
        { 
            // This is tricky. Consider the example. 
            // AAACAAAA and i = 7. The idea is similar 
            // to search step. 
            if (len != 0) { 
                len = lps[len - 1]; 
  
                // Also, note that we do not increment 
                // i here 
            } 
            else // if (len == 0) 
            { 
                lps[i] = 0; 
                i++; 
            } 
        } 
    } 
} 
  
// Driver program to test above function 
int main() 
{ 
    char txt[] = "ABABDABACDABABCABAB"; 
    char pat[] = "ABABCABAB"; 
    KMPSearch(pat, txt); 
    return 0; 
} 

```

## 使用哈希查找和strstr速度比较
hash在查找字符串时，每次偏移1个字节，并计算hash, 为了不重复计算重叠部分，有个算法'Karp-Rabin with Rolling Hash'。 但是速度都不如glibc的 strstr， 因为这里的hash函数没有用到硬件加速，但是strstr用了。

```c
#define BASE 256         // Alphabet size (ASCII)
#define MOD 101          // A prime number to avoid overflow

// Rolling hash version of strstr()
const char* rabin_karp_strstr(const char* haystack, const char* needle) {
    int n = strlen(haystack);
    int m = strlen(needle);

    if (m == 0) return haystack;
    if (m > n) return NULL;

    int i;
    int hash_needle = 0;  // Hash for needle
    int hash_window = 0;  // Hash for current window in haystack
    int h = 1;

    // The value of h is BASE^(m-1) % MOD
    for (i = 0; i < m - 1; i++)
        h = (h * BASE) % MOD;

    // Calculate the hash value of needle and first window
    for (i = 0; i < m; i++) {
        hash_needle  = (BASE * hash_needle + needle[i]) % MOD;
        hash_window = (BASE * hash_window + haystack[i]) % MOD;
    }

    // Slide the pattern over text
    for (i = 0; i <= n - m; i++) {
        if (hash_needle == hash_window) {
            // Verify characters to avoid false positive
            if (strncmp(&haystack[i], needle, m) == 0)
                return &haystack[i];
        }

        // Calculate hash for next window
        if (i < n - m) {
            hash_window = (BASE * (hash_window - haystack[i] * h) + haystack[i + m]) % MOD;
            if (hash_window < 0)
                hash_window += MOD;  // Ensure non-negative
        }
    }

    return NULL;
}
```