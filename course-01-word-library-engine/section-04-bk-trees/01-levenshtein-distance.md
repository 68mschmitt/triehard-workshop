# Levenshtein Distance

## Why You Need This

Your requirements say: "Use edit distance (e.g., Levenshtein) as the similarity metric." When a user types "helo", you need to suggest "hello". Levenshtein distance tells you how "close" two strings are.

## What to Learn

### The Intuition

Levenshtein distance = minimum number of single-character edits to transform one string into another.

Edit operations:
1. **Insert** a character
2. **Delete** a character  
3. **Substitute** one character for another

```
"helo" → "hello"  (insert 'l')       = distance 1
"wrld" → "world"  (insert 'o')       = distance 1
"cat"  → "car"    (substitute t→r)  = distance 1
"cat"  → "cats"   (insert 's')      = distance 1
"cat"  → "dog"    (c→d, a→o, t→g)  = distance 3
```

### The Classic Algorithm

Dynamic programming on a 2D matrix:

```c
int levenshtein(const char *s1, const char *s2) {
    int len1 = strlen(s1);
    int len2 = strlen(s2);
    
    // Matrix: dp[i][j] = distance between s1[0..i-1] and s2[0..j-1]
    int **dp = malloc((len1 + 1) * sizeof(int *));
    for (int i = 0; i <= len1; i++) {
        dp[i] = malloc((len2 + 1) * sizeof(int));
    }
    
    // Base cases
    for (int i = 0; i <= len1; i++) dp[i][0] = i;  // Delete all
    for (int j = 0; j <= len2; j++) dp[0][j] = j;  // Insert all
    
    // Fill matrix
    for (int i = 1; i <= len1; i++) {
        for (int j = 1; j <= len2; j++) {
            int cost = (s1[i-1] == s2[j-1]) ? 0 : 1;
            dp[i][j] = min3(
                dp[i-1][j] + 1,      // Delete from s1
                dp[i][j-1] + 1,      // Insert into s1
                dp[i-1][j-1] + cost  // Substitute (or match)
            );
        }
    }
    
    int result = dp[len1][len2];
    
    // Free matrix
    for (int i = 0; i <= len1; i++) free(dp[i]);
    free(dp);
    
    return result;
}
```

### Visualizing the Matrix

For "cat" → "car":
```
      ""  c  a  r
  "" [ 0  1  2  3 ]
  c  [ 1  0  1  2 ]
  a  [ 2  1  0  1 ]
  t  [ 3  2  1  1 ]  ← Answer: 1
```

Each cell = min of:
- Cell above + 1 (delete)
- Cell left + 1 (insert)  
- Cell diagonal + 0/1 (match/substitute)

### Space Optimization

Full matrix uses O(m×n) space. You only need two rows:

```c
int levenshtein_optimized(const char *s1, const char *s2) {
    int len1 = strlen(s1);
    int len2 = strlen(s2);
    
    int *prev = malloc((len2 + 1) * sizeof(int));
    int *curr = malloc((len2 + 1) * sizeof(int));
    
    for (int j = 0; j <= len2; j++) prev[j] = j;
    
    for (int i = 1; i <= len1; i++) {
        curr[0] = i;
        for (int j = 1; j <= len2; j++) {
            int cost = (s1[i-1] == s2[j-1]) ? 0 : 1;
            curr[j] = min3(
                prev[j] + 1,
                curr[j-1] + 1,
                prev[j-1] + cost
            );
        }
        int *tmp = prev; prev = curr; curr = tmp;
    }
    
    int result = prev[len2];
    free(prev);
    free(curr);
    return result;
}
```

O(min(m,n)) space instead of O(m×n). Big win for long strings.

### Early Termination

For suggestions, you often have a maximum distance threshold:

```c
int levenshtein_bounded(const char *s1, const char *s2, int max_dist) {
    int len1 = strlen(s1);
    int len2 = strlen(s2);
    
    // Quick rejection
    if (abs(len1 - len2) > max_dist) return max_dist + 1;
    
    int *prev = calloc(len2 + 1, sizeof(int));
    int *curr = calloc(len2 + 1, sizeof(int));
    
    for (int j = 0; j <= len2; j++) prev[j] = j;
    
    for (int i = 1; i <= len1; i++) {
        curr[0] = i;
        int row_min = curr[0];
        
        for (int j = 1; j <= len2; j++) {
            int cost = (s1[i-1] == s2[j-1]) ? 0 : 1;
            curr[j] = min3(prev[j] + 1, curr[j-1] + 1, prev[j-1] + cost);
            if (curr[j] < row_min) row_min = curr[j];
        }
        
        // Early termination: entire row exceeds threshold
        if (row_min > max_dist) {
            free(prev); free(curr);
            return max_dist + 1;
        }
        
        int *tmp = prev; prev = curr; curr = tmp;
    }
    
    int result = prev[len2];
    free(prev); free(curr);
    return result;
}
```

If the user's word is 5 characters and your dictionary word is 50, you can skip without full computation.

### Time Complexity

- Basic: O(m × n)
- With early termination: O(m × min(n, m + max_dist))

For typical word lengths (5-15 chars) and small max_dist (1-3), this is fast.

## Where to Learn

1. **Wikipedia "Levenshtein distance"** - Good explanation with examples
2. **"Introduction to Algorithms"** - Edit distance / sequence alignment
3. **LeetCode 72** - "Edit Distance" problem
4. **"Algorithms on Strings" by Crochemore** - Deep dive

## Practice Exercise

Implement:
```c
int levenshtein(const char *s1, const char *s2);
int levenshtein_bounded(const char *s1, const char *s2, int max_dist);
```

Test cases:
- `levenshtein("kitten", "sitting")` → 3
- `levenshtein("", "hello")` → 5
- `levenshtein("hello", "hello")` → 0
- `levenshtein_bounded("cat", "catastrophe", 2)` → 3 (exceeds bound)

Benchmark: 100,000 comparisons of ~10 char words should take < 1 second.

## Connection to Project

This is your similarity metric. When the user types "helo":
1. BK-tree finds candidate words within distance 2
2. Each candidate's exact distance is computed
3. Results sorted by distance
4. "hello" (distance 1) ranks above "hero" (distance 2)

The bounded version is crucial - you'll call this thousands of times per suggestion query.
