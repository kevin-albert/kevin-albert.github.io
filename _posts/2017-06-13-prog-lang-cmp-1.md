---
layout:     post
title:      "Programming Languages Compared - Part 1"
date:       2017-06-13 19:43:00
summary:    "Levenshtein distance in javascript, haskell, and assembly"
categories: things stuff
---

This is my first (of many, hopefully) blog posts where I take one thing and implement it in multiple languages. The goal is not to try and solve the problem as efficiently as possible, or with the smallest amount of source code. Instead, I hope to use these posts to compare and contrast features of the given languages. For my first language comparison, I've chosen to solve a simple problem using a [known algorithm](https://en.wikipedia.org/wiki/Levenshtein_distance#Iterative_with_two_matrix_rows) in three languages: Javascript, Haskell, and x86 (64-bit / Intel / macOS), using only their respective standard libraries. 

## The Problem 

The problem is to produce a function which, given two strings `a` and `b`, of length `n` and `m` returns the number of edits (insertions, deletions, or replacements) required to change `a` into `b` (or vice-versa). For example, `levenshtein("espresso", "expreso")` returns 2 - one replacement and one insertion. This distance can be expressed as the recurrence:
\begin{align}
Lev(i,j) = \cases{
    \max(i,j) & i = 0 or j = 0 \cr
    \min(1+Lev(i-1,j), \cr
        \quad 1+Lev(i,j-1),\cr
        \quad Lev(i-1,j-1) + (i == j)) & \text{otherwise}
}
\end{align}
which returns the levenshtein distance from `a[1..i]` to `b[1..j]`, so `Lev(n, m)` is the final answer. This can be computed recursively in exponential time. However, it can be memoized using a matrix with `n` rows and `m` columns in $$\Theta(nm)$$ time and $$\Theta(nm)$$ space, and further optimized to use only $$\Theta(m)$$ space, by throwing away unneeded matrix rows. So, while my goal is not to provide optimized code, I decided that each solution should solve the problem in $$\Theta(mn)$$ time and $$\Theta(m)$$ space.  

This problem seems a bit unfair from the outset - procedural and imperative languages (Javascript, x86) lend themselves very well to implementing well-known algorithms, while declarative languages (Haskell) don't. The known algorithm (in this case, space / time efficient levenshtein distance) maps directly to a sequence of instructions that can be evaluated in an imperative context. With an imperative language, you tell the computer _how_ to solve a problem by providing it with an algorithm, but with a declarative language like Haskell or SQL you tell the computer what you want and let it decide what to do. With Haskell, we could fake it &ndash; use mutable arrays and translate the loops into tail recursions, etc. In order to capitalize on the strengths of the language, we need to decompose it into subproblems whose solutions can be queried rather than computed. 

## Solutions
All of that being said, the shortest solution here is actually the Haskell version, and it still follows the algorithm pretty closely. It defines two inner functions - one to recursively build a single matrix row, and another to build up the matrix, accepting the previous row as an argument and recursing down to create the next one until it reaches the end.  

The Javascipt solution is rather brief as well, and we get to make use of its array destructuring in two places - at the beginning, where we assign `n` and `m` to string lengths and at the end where we swap the two arrays.  

Implementing the function in x86 was rather straightforward &ndash; two arrays of length `m` are pushed onto the stack, and then swapped at the end of each outer loop. We make use of `cmovne` and `cmovg` for conditional register assignments, which is nice for storing the substitution cost (`cost = a[i] == a[j] ? 0 : 1`) and then taking the min of three registers while computing the value of the recurrence. Not surprisingly, it is just over 100 lines.  

Without further ado, here's the code &ndash; you can also find it on [Github](https://gist.github.com/kevin-albert/df742e837ac7e4247bfe6834d834ae6b). These snippets would be presented side-by-side, but I don't know how to do that. 

### Javascript
```js
/**
 * Compute according to recurrence:
 * 
 *            / max(i,j):                       i = 0 or j = 0
 * Lev(i,j) = | min(Lev(i-1,j)+1,               otherwise
 *            |     Lev(i,j-1)+1,
 *            \     Lev(i-1,j-1)+(a[i] = b[j]))
 * 
 * @param a a string
 * @param b another string 
 */
function levenshtein(a, b) {
    let [n, m] = [a.length, b.length];
    let r0 = new Uint32Array(m + 1),
        r1 = new Uint32Array(m + 1);

    for (let j = 0; j <= m; ++j)
        // edit distance at M[0,j] is from '' to b[0...j]
        r0[j] = j;

    for (let i = 0; i < n; ++i) {
        // edit distance at M[i+1,0] is from a[0...i] to ''
        r1[0] = i + 1;

        // compute row according to recurrence
        for (let j = 0; j < m; ++j) {
            let cost = (a[i] == b[j]) ? 0 : 1;
            r1[j + 1] = Math.min(r1[j] + 1, r0[j + 1] + 1, r0[j] + cost);
        }

        // switch previous row to current
        [r1, r0] = [r0, r1];
    }

    return r0[m];
}
```

### Haskell

```hs
{-|
  Compute according to recurrence:
  
             / max(i,j):                        i = 0 or j = 0
  Lev(i,j) = | min(Lev(i-1,j)+1,                otherwise
             |     Lev(i,j-1)+1,
             \     Lev(i-1,j-1)+(a[i] = b[j]))
-}
levenshtein :: String -> String -> Int
levenshtein a b = last (rows 0 a [0..length b])
    where 
    -- build a row, then recurse down with i+1 for the next
    rows i [] r = r
    rows i (a':sa) r = rows (i+1) sa ((i+1):(cols a' b r (i+1)))
    
    -- builds a single row, left to right
    cols _ "" [p] _ = []
    cols a' (b':sb) (r0:r1:r) costPrev =
        minCost : (cols a' (sb) (r1:r) minCost)
        where                   -- we compute the edit distance at this point
            cost| a' == b'  = 0
                | otherwise = 1;
            minCost = min (costPrev + 1) (min (r1 + 1) (r0 + cost));
```

### x86
```nasm
section .text
extern _strlen
global _levenshtein

;
; int levenshtein(char *a, char *b); 
; 
; Compute according to recurrence:
; 
;            / max(i,j):                        i = 0 or j = 0
; Lev(i,j) = | min(Lev(i-1,j)+1,                otherwise
;            |     Lev(i,j-1)+1,
;            \     Lev(i-1,j-1)+(a[i] = b[j]))
; 
_levenshtein:
    push        rbp
    mov         rbp, rsp
    sub         rsp, 32
    mov         [rbp-8], rdi    ; push a and b, get ready to call strlen
    mov         [rbp-16], rsi 

    call        _strlen         ; strlen(a)
    mov         [rbp-20], eax 

    mov         rdi, [rbp-16]   ; strlen(b)
    call        _strlen

    mov         rdi, [rbp-8]    ; rdi = a
    mov         rsi, [rbp-16]   ; rsi = b
    mov         ebx, eax        ; ebx = strlen(b)
    mov         eax, [rbp-20]   ; eax = strlen(a)

    xor         rcx, rcx        ; now, allocate two arrays of size (ebx+1)*4
    mov         ecx, ebx        ; set ecx = ebx + 1, then shift left by 2
    add         ecx, 1 
    shl         ecx, 2
    mov         r15, rcx        ; use r15 to store our 16 byte aligned size
    shl         r15, 1          ; alignment doesn't matter right now; our stack
    add         r15, 15         ; is currently at a local maxima. however if we
    and         r15, -16        ; want to call any other functions (eg printf
    mov         r8, rsp         ; for debugging) then this is nice to have 
    sub         r8, rcx         ; built in.
    mov         r9, r8          ; r8 and r9 are our arrays corresponding to 
    sub         r9, rcx         ; the previous and current matrix row
    sub         rsp, r15
    push        r15

    xor         rdx, rdx        ; fill previous row with 0,1,2,...,m
    jmp         end_init_cols   ; because our edit distance there corresponds
loop_init_cols:                 ; to when a is empty and b has size 0,1,2,...,m 
    mov         [r8+rdx*4], edx
    inc         edx
end_init_cols:
    cmp         edx, ebx
    jle loop_init_cols
                                ; outer loop (iterate through rows)
    xor         rcx, rcx        ; for (i = 0; i < n; ++i)
    jmp         end_rows
loop_rows:
    mov         [r9], ecx       ; edit distance here is when b is empty and a 
    add word    [r9], 1         ; has i characters

                                ; inner loop (iterate through columns)
    xor         rdx, rdx        ; for (j = 0; j < m; ++j)
    jmp         end_cols
loop_cols:                      ; finally, this is where we calculate the
                                ; recurrence

    xor         r12, r12        ; see if a[i] == b[j]
    mov         r13d, 1         ; set substitution cost to 0 or 1 accordingly
    mov         r10b, [rdi+rcx]
    mov         r11b, [rsi+rdx]
    cmp         r10b, r11b
    cmovne      r12d, r13d

    mov         r13d, [r9+rdx*4]; option A
    add         r13d, 1         ; r1[j]+1 - delete one char from b

                                
    mov         r14d, [r8+(rdx+1)*4] ; option B
    add         r14d, 1         ; r0[j+1]+1 - delete one char from a

    mov         r15d, [r8+rdx*4]; option C
    add         r15d, r12d      ; r0[j]+cost - replacement (if needed)

    cmp         r13d, r14d      ; take the min of the three options
    cmovg       r13d, r14d 
    cmp         r13d, r15d 
    cmovg       r13d, r15d

    inc         edx             ; r1[j + 1] = min (inc first to save an add)
    mov         [r9+(rdx*4)], r13d

end_cols:                       ; end inner loop
    cmp         edx, ebx
    jl          loop_cols

    mov         r10, r8         ; swap rows, so current becomes previous
    mov         r8, r9 
    mov         r9, r10

    inc         ecx
end_rows:                       ; end outer loop
    cmp         ecx, eax
    jl          loop_rows
    
    mov         eax, [r8+rbx*4] ; result is at the end of the last row


    pop         r15             ; free our arrays
    add         rsp, r15 

    add         rsp, 32         ; fix up the stack and return
    pop         rbp
    ret
```