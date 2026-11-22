# CSV 報表

把一份小型銷售 CSV 寫入暫存檔，再逐行讀回，並印出欄位對齊的各地區摘要表。

## 程式碼

```nabi
// Write a small sales CSV, read it back line by line, and print a per-region report.
use std::env;
use std::fs;
use std::fs::File;
use core::io::{IoError, BufReader, BufWriter};

const ROWS: str = "date,region,product,units,unit_price
2026-01-03,north,widget,12,2.50
2026-01-03,south,gadget,3,19.99
2026-01-04,north,gadget,1,19.99
2026-01-05,east,widget,40,2.25
2026-01-05,south,widget,7,2.50
2026-01-06,west,gizmo,two,7.00
2026-01-07,east,gizmo,5,7.00
2026-01-08,north,gizmo,2,7.50
2026-01-09,east;widget,3,2.25";

struct Sale {
    region: str;
    units: i64;
    price: f64;
}

struct Group {
    orders: i64 = 0;
    units: i64 = 0;
    revenue: f64 = 0.0;
}

enum RowError {
    Fields(usize),
    Number(str),
}

fn parse_row(line: str) throws (RowError) -> Sale {
    let f = line.split(",").map(|part| part.trim()).to_list();
    if f.len() != 5 { throw RowError::Fields(f.len()); }
    let units = try f[3].parse::<i64>() catch _ { throw RowError::Number(f[3]); };
    let price = try f[4].parse::<f64>() catch _ { throw RowError::Number(f[4]); };
    Sale { region: f[1], units: units, price: price }
}

fn write_csv(path: str) throws (IoError) {
    using w = BufWriter::new(try File::create(path));
    for line in ROWS.lines() {
        try w.write_all(f"{line}\n");
    }
}

fn summarize(path: str) throws (IoError) -> Dict<str, Group> {
    let groups = Dict::new();
    using r = BufReader::new(try File::open(path)) {
        let n = 0;
        try for line in r.lines() {
            n += 1;
            if n == 1 || line.is_empty() { continue; } // header
            let sale = try parse_row(line) catch e: RowError {
                match e {
                    RowError::Fields(k) => println(f"line {n}: expected 5 fields, got {k}, skipped"),
                    RowError::Number(s) => println(f"line {n}: bad number '{s}', skipped"),
                }
                continue;
            };
            groups[sale.region, default: Group {}].orders += 1;
            groups[sale.region].units += sale.units;
            groups[sale.region].revenue += sale.units as f64 * sale.price;
        }
    }
    groups
}

fn print_report(groups: Dict<str, Group>) {
    let regions = groups.keys().sorted();

    println(f"{"region":<8}|{"orders":>7}|{"units":>6}|{"revenue":>9}|{"avg":>7}");
    println(f"{"":-<8}+{"":-<7}+{"":-<6}+{"":-<9}+{"":-<7}");
    let total = Group {};
    for name in regions {
        let g = groups[name];
        let avg = g.revenue / g.units as f64;
        println(f"{name:<8}|{g.orders:>7}|{g.units:>6}|{g.revenue:>9.2}|{avg:>7.2}");
        total.orders += g.orders;
        total.units += g.units;
        total.revenue += g.revenue;
    }
    println(f"{"total":<8}|{total.orders:>7}|{total.units:>6}|{total.revenue:>9.2}|{"":>7}");
}

fn main() throws (IoError) {
    let path = (env::temp_dir() / "nabi-sales-demo.csv").as_str();
    try write_csv(path);
    let groups = try summarize(path) finally { try! fs::remove_file(path); };
    print_report(groups);
}
```

```output
line 7: bad number 'two', skipped
line 10: expected 5 fields, got 4, skipped
region  | orders| units|  revenue|    avg
--------+-------+------+---------+-------
east    |      2|    45|   125.00|   2.78
north   |      3|    15|    64.99|   4.33
south   |      2|    10|    77.47|   7.75
total   |      7|    70|   267.46|
```

## 說明

- 檔案建立在 `env::temp_dir()` 之下，並在 `main` 的 `finally` 中刪除，所以即使彙總失敗也不會留下暫存檔（[檔案與 I/O](../learn/13-io.html)、[`env::temp_dir`](../../std/std/env.html#fn.temp_dir)）。
- 寫入透過 `BufWriter`，讀取透過 `BufReader`，兩者都以 `using` 繫結（[`using`](../learn/10-errors.html)）：`write_csv` 中的陳述式 `using w = ..;` 在函式區塊結束時關閉檔案，`summarize` 中的區塊形式則在其區塊結束時關閉，而寫入器會先把手上剩下的資料寫出，所以寫入失敗永遠不會被漏掉。`r.lines()` 是可能失敗的迭代器，所以 `try for` 逐一取出每一行，並把讀取錯誤往外傳（[`BufReader`](../../std/core/io.html#struct.BufReader)、[`fs`](../../std/std/fs.html)）。
- `parse_row` 遇到格式錯誤的資料列會擲回 `RowError`；呼叫端在 `catch` 區塊中印出警告，再以 `continue` 繼續。一行壞資料不會毀掉整份報表（[錯誤](../learn/10-errors.html)）。
- 總計是 `Dict<str, Group>`，型別由 `summarize` 的回傳型別決定。地區不存在時，`groups[region, default: Group {}].orders += 1` 會建立新項目，後面幾行則就地累加到已存的項目上。`Group` 的每個欄位都有預設值，所以單寫 `Group {}` 就從零開始（[集合型別與迭代器](../learn/08-collections-iterators.html)）。
- 格式規範讓表格對齊：`{name:<8}` 靠左對齊，`{g.revenue:>9.2}` 使用寬度 9 和兩位小數，`{"":-<8}` 則用 `-` 填滿空字串，畫出分隔線（[2.10.8 對齊與寬度](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)）。
