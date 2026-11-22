# 单词计数

统计一段文本的行数、单词数和字符数，再用 `Dict` 和排序选出出现最多的五个单词。

## 代码

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

## 讲解

- 计数由 `str` 上的迭代器完成。`lines()` 逐行产出，`split_whitespace()` 逐个产出单词，`count()` 统计个数。字符数是 `char_count()`，而不是字节长度 `len()`（[字符串](../learn/09-strings.html)，[`str` 的方法](../../std/core/str.html)）。
- `normalize` 在 `StrBuf` 中逐个字符地构造单词，去掉标点并转为小写，所以 `"Dog,"` 和 `"dog."` 算作同一个单词（[`StrBuf`](../../std/core/str.html#struct.StrBuf)）。
- 频率表是 `Dict<str, i32>`。`counts[w, default: 0] += 1` 在 `w` 不存在时先存入 `0`，再加一（[集合与迭代器](../learn/08-collections-iterators.html)，[`Dict`](../../std/core/collections/dict.html#struct.Dict)）。
- 对字典调用 `sorted_by_key` 会把它的 `(word, count)` 对作为排好序的列表给出。键 `(-c, w)` 让计数最高的排在最前，计数相同时按字母顺序排列，所以输出总是相同的；`take(n).to_list()` 保留前 `n` 个。`enumerate(start: 1)` 从 1 开始给名次编号（[集合与迭代器](../learn/08-collections-iterators.html)）。
- 表格用 f-string 的格式说明符对齐：`{w:<6}` 左对齐，`{c:>2}` 右对齐（[2.10.8 对齐与宽度](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)）。
