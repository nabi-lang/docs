# 字數統計

計算一段文字的行數、單字數與字元數，再用 `Dict` 和排序挑出出現次數最多的五個單字。

## 程式碼

```nabi
// Count lines, words and chars, then list the most frequent words.

const TEXT: str = "The quick brown fox jumps over the lazy dog.
The dog sleeps; the fox does not.
A fox, a dog, and the long summer day.

Quick thinking wins the day!";

struct Stats {
    lines: usize;
    words: usize;
    chars: usize;
}

fn stats(text: str) -> Stats {
    let s = Stats { lines: 0, words: 0, chars: text.char_count() };
    for line in text.lines() {
        s.lines += 1;
        s.words += line.split_whitespace().count();
    }
    s
}

// Lowercase a token and drop surrounding punctuation ("Dog," -> "dog").
fn normalize(token: str) -> str {
    let sb = StrBuf::new();
    for c in token.chars() {
        if c.is_alphanumeric() || c == '\'' {
            sb.push_str(c.to_lowercase());
        }
    }
    sb.to_str()
}

fn frequencies(text: str) -> Dict<str, i32> {
    let counts: Dict<str, i32> = Dict::new();
    for token in text.split_whitespace() {
        let w = normalize(token);
        if w.is_empty() { continue; }
        counts[w, default: 0] += 1;
    }
    counts
}

fn top(counts: Dict<str, i32>, n: usize) -> List<(str, i32)> {
    // Most frequent first; ties broken alphabetically so output is stable.
    counts.sorted_by_key(|(w, c)| (-c, w)).take(n).to_list()
}

fn main() {
    let s = stats(TEXT);
    println(f"lines {s.lines:>4}");
    println(f"words {s.words:>4}");
    println(f"chars {s.chars:>4}");

    let counts = frequencies(TEXT);
    println(f"\n{counts.len()} distinct words, top 5:");
    for (rank, (w, c)) in top(counts, 5).enumerate(start: 1) {
        let bar = "#".repeat(c as usize);
        println(f"{rank}. {w:<6} {c:>2} {bar}");
    }
}
```

```output
lines    5
words   30
chars  147

18 distinct words, top 5:
1. the     6 ######
2. dog     3 ###
3. fox     3 ###
4. a       2 ##
5. day     2 ##
```

## 說明

- 計數由 `str` 的迭代器負責。`lines()` 逐行產出，`split_whitespace()` 逐字產出，`count()` 計算數量。字元數是 `char_count()`，而不是位元組長度 `len()`（[字串](../learn/09-strings.html)、[`str` 方法](../../std/core/str.html)）。
- `normalize` 在 `StrBuf` 中一次一個字元地組出單字，同時去掉標點並轉成小寫，所以 `"Dog,"` 和 `"dog."` 算作同一個單字（[`StrBuf`](../../std/core/str.html#struct.StrBuf)）。
- 頻率表是 `Dict<str, i32>`。`w` 不存在時，`counts[w, default: 0] += 1` 會先存入 `0`，再加一（[集合型別與迭代器](../learn/08-collections-iterators.html)、[`Dict`](../../std/core/collections/dict.html#struct.Dict)）。
- 對字典呼叫 `sorted_by_key`，會得到排序好的 `(word, count)` 配對串列。鍵 `(-c, w)` 讓次數最多的排在前面，次數相同時依字母順序排列，所以輸出永遠相同；`take(n).to_list()` 保留前 `n` 個。`enumerate(start: 1)` 從 1 開始為名次編號（[集合型別與迭代器](../learn/08-collections-iterators.html)）。
- 表格用 f-string 的格式規範對齊：`{w:<6}` 靠左對齊，`{c:>2}` 靠右對齊（[2.10.8 對齊與寬度](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)）。
