# CSV 报表

把一份小的销售 CSV 写入临时文件，再逐行读回，并输出一张列对齐的按地区汇总表。

## 代码

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

## 讲解

- 文件创建在 `env::temp_dir()` 下，并在 `main` 的 `finally` 中删除，所以即使汇总失败也不会留下临时文件（[文件与 I/O](../learn/13-io.html)，[`env::temp_dir`](../../std/std/env.html#fn.temp_dir)）。
- 写入经过 `BufWriter`，读取经过 `BufReader`，两者都用 `using` 绑定（[`using`](../learn/10-errors.html)）：`write_csv` 中的语句 `using w = ..;` 在函数的块结束时关闭文件，`summarize` 中的块形式在它的块结束时关闭文件；writer 会先写出仍持有的内容，所以写入失败不会被遗漏。`r.lines()` 是可能失败的迭代器，所以由 `try for` 逐行取出，并把读取错误向外传递（[`BufReader`](../../std/core/io.html#struct.BufReader)，[`fs`](../../std/std/fs.html)）。
- 遇到格式错误的行时，`parse_row` 抛出 `RowError`；调用方在 `catch` 块中输出警告，并用 `continue` 继续。一行坏数据不会毁掉整个报表（[错误处理](../learn/10-errors.html)）。
- 汇总结果是 `Dict<str, Group>`，类型由 `summarize` 的返回类型确定。地区不存在时，`groups[region, default: Group {}].orders += 1` 会新建一个条目，之后的几行就地累加到存储的条目上。`Group` 的每个字段都有默认值，所以单独的 `Group {}` 从零开始（[集合与迭代器](../learn/08-collections-iterators.html)）。
- 格式说明符让表格对齐：`{name:<8}` 左对齐，`{g.revenue:>9.2}` 使用宽度 9 和两位小数，`{"":-<8}` 用 `-` 填充空字符串来画分隔线（[2.10.8 对齐与宽度](../spec/refs-standard-libraries/200-core/210-fmt.html#2108-align--fill--width--precision)）。
