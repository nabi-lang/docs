# 1. 첫 데이터 분석

이 튜토리얼은 작은 데이터 집합을 CSV 파일에서 숫자와 그래프까지 가져갑니다. `table`로 읽고, 정리하고, 그룹으로 묶어 요약하고, `stats`로 차이를 검정하고, `plot`으로 그립니다. 단계마다 완결된 스크립트이므로 어느 것이든 `nabi run file.nabi`로 실행할 수 있습니다.

`table`, `stats`, `plot`은 공식 패키지라서 스크립트가 `nabi add` 없이 쓸 수 있습니다. 처음 `nabi run`을 하면 찾아낸 패키지를 하나씩 추가하고 그렇다고 알려 줍니다(`Added table ^0.1 (official)`).

## CSV 파일 읽기

데이터는 기상 관측소 세 곳의 한 주 동안의 일별 관측값입니다. `table::read_csv`는 파일을 읽고 값을 보고 열마다 타입을 정합니다. `table::read_csv_str`은 이미 메모리에 있는 텍스트로 같은 일을 합니다. 여기서는 스크립트가 먼저 파일을 써 두므로 어디서든 실행됩니다.

```nabi
use std::{env, fs, process};

let csv = "day,station,temp,rain
1,north,12.5,3.0
1,south,18.0,0.0
1,east,15.0,0.5
2,north,13.5,1.5
2,south,19.5,0.0
2,east,14.0,2.0
3,north,11.0,
3,south,17.5,4.5
3,east,15.5,0.0
4,north,12.0,6.0
4,south,20.5,0.0
4,east,16.0,1.0
";
let path = env::temp_dir() / f"weather_{process::id()}.csv";
try fs::write(path, csv);

let t = try table::read_csv(path.as_str());
println(t.shape());
println(t.head(4));
try fs::remove_file(path);
```

```output
(12, 4)
shape: (4, 4)
day | station | temp | rain
i64 | str     | f64  | f64
----+---------+------+-----
  1 | north   | 12.5 |  3.0
  1 | south   | 18.0 |  0.0
  1 | east    | 15.0 |  0.5
  2 | north   | 13.5 |  1.5
```

머리글 행이 열 이름을 정하고, 열마다 타입이 하나입니다. `day`는 정수, `station`은 텍스트, `temp`와 `rain`은 부동소수점 수입니다. 3일째의 빈 `rain`은 null, 곧 없는 값입니다. 표는 null을 0이나 NaN으로 바꾸지 않고 null로 둡니다.

## 정리하기

평균이나 검정을 내기 전에 null을 어떻게 할지 정해야 합니다. `fill_null`은 그 자리에 값을 넣고, `drop_nulls`는 null이 있는 행을 지웁니다. 어느 쪽이 맞는지는 데이터에 달렸습니다. 우량계 값이 빠졌다고 "비가 오지 않았다"는 뜻은 아니므로 이 스크립트는 그 행을 지우고, 두 경우의 개수를 모두 보여 줍니다.

```nabi
let t = try table::read_csv_str("day,station,temp,rain
1,north,12.5,3.0
2,north,13.5,1.5
3,north,11.0,
4,north,12.0,6.0
");
let rain = try t.column("rain");
println(f"{rain.null_count()} null of {t.nrows()}");

let dropped = try t.drop_nulls(["rain"]);
let filled = try t.fill_null("rain", table::Value::Float(0.0));
println(f"dropped: {dropped.nrows()} rows; filled: {try filled.column("rain").to_f64()}");
```

```output
1 null of 4
dropped: 3 rows; filled: [3.0, 1.5, 0.0, 6.0]
```

## 열 더하기와 행 거르기

열 표현식은 배열과 같은 점 연산자(`.+`, `.*`, `.>` ..)를 씁니다. `col("temp") .* 1.8 .+ 32.0`은 `temp`로부터 행마다 계산하는 새 열입니다. `filter`는 조건이 성립하는 행만 남깁니다.

```nabi
use table::col;

let t = table::table!{
    "station": ["north", "south", "east", "north", "south", "east"],
    "temp":    [12.5, 18.0, 15.0, 13.5, 19.5, 14.0],
    "rain":    [3.0, 0.0, 0.5, 1.5, 0.0, 2.0],
};
let t = try t.with_column("temp_f", col("temp") .* 1.8 .+ 32.0);
let wet = try t.filter(col("rain") .> 1.0);
println(wet);
```

```output
shape: (3, 4)
station | temp | rain | temp_f
str     | f64  | f64  | f64
--------+------+------+-------
north   | 12.5 |  3.0 |   54.5
north   | 13.5 |  1.5 |   56.3
east    | 14.0 |  2.0 |   57.2
```

## 그룹으로 묶어 요약하기

`group_by`는 관측소마다 행을 모으고, `agg`는 받은 표현식마다 그룹당 값 하나를 계산합니다. 집계 결과는 열 이름을 따서 이름이 붙고(`temp_mean`), `.named`로 이름을 바꿀 수 있습니다.

```nabi
use table::{desc, max, mean, sum};

let t = table::table!{
    "station": ["north", "south", "east", "north", "south", "east", "north", "south", "east"],
    "temp":    [12.5, 18.0, 15.0, 13.5, 19.5, 14.0, 11.0, 17.5, 15.5],
    "rain":    [3.0, 0.0, 0.5, 1.5, 0.0, 2.0, 0.5, 4.5, 0.0],
};
let by = try (try t.group_by(["station"])).agg([mean("temp"), max("temp"), sum("rain").named("total_rain")]);
println(try by.sort([desc("temp_mean")]));
```

```output
shape: (3, 4)
station | temp_mean | temp_max | total_rain
str     | f64       | f64      | f64
--------+-----------+----------+-----------
south   | 18.333333 |     19.5 |        4.5
east    | 14.833333 |     15.5 |        2.5
north   | 12.333333 |     13.5 |        5.0
```

## 열에 대한 통계

열은 `to_f64`로 `Vector<f64>`가 되고, 거기서부터는 `stats`의 모든 함수를 쓸 수 있습니다. `stats::describe`는 흔히 보는 요약을 한 번에 줍니다.

```nabi
let t = table::table!{
    "station": ["north", "south", "north", "south", "north", "south", "north", "south"],
    "temp":    [12.5, 18.0, 13.5, 19.5, 11.0, 17.5, 12.0, 20.5],
};
let temp = try t.column("temp").to_f64();
println(f"mean {stats::mean(temp):.2}, std {stats::std(temp, ddof: 1):.2}, median {stats::median(temp)}");

let s = stats::describe(temp);
println(f"n {s.n}, min {s.min}, q1 {s.q1}, q3 {s.q3}, max {s.max}");
```

```output
mean 15.56, std 3.72, median 15.5
n 8, min 11, q1 12.375, q3 18.375, max 20.5
```

`ddof: 1`은 `n - 1`로 나누는 표본 표준편차를 뜻합니다. 이것이 없으면 `std`는 NumPy처럼 `n`으로 나눕니다.

## 차이가 진짜일까?

남쪽 관측소가 더 따뜻해 보입니다. 두 표본 t-검정은 두 관측소의 평균이 같다면 이만큼 큰 차이가 나올 가능성이 얼마인지 묻습니다. `stats::ttest_ind`는 분산이 같다고 가정하지 않는 Welch 검정을 수행하며, 결과에는 통계량, p-값, 차이의 신뢰 구간이 들어 있습니다.

```nabi
use table::col;

let t = table::table!{
    "station": ["north", "south", "north", "south", "north", "south", "north", "south"],
    "temp":    [12.5, 18.0, 13.5, 19.5, 11.0, 17.5, 12.0, 20.5],
};
let north = try (try t.filter(col("station") .== "north")).column("temp").to_f64();
let south = try (try t.filter(col("station") .== "south")).column("temp").to_f64();

let r = try stats::ttest_ind(south, north);
println(f"t = {r.statistic:.2}, p = {r.pvalue:.5}");
println(f"south is warmer by {r.estimate:.2} C, 95% interval {r.ci(0.95)}");
```

```output
t = 7.68, p = 0.00036
south is warmer by 6.62 C, 95% interval [4.474617987003727, 8.775382012996273]
```

p-값이 이만큼 작다는 것은 평균이 같은 두 관측소에서 이렇게 멀리 떨어진 표본이 나오는 일은 드물다는 뜻입니다. 관측값이 넷씩뿐이라 구간은 아직 넓습니다. 데이터가 많아지면 좁아집니다.

## 그래프 그리기

`plot`은 메서드 호출로 레이어를 쌓아 그림을 만들고, SVG, PNG, 터미널용 텍스트(`ax.term()`)로 씁니다. `scatter`와 `line`은 통계에 쓴 것과 같은 벡터를 받고, `label`은 범례에 쓸 레이어 이름을 붙입니다. `save`는 파일 이름을 보고 형식을 고릅니다.

```nabi
use std::{env, fs, process};

let day = [1.0, 2.0, 3.0, 4.0];
let north = [12.5, 13.5, 11.0, 12.0];
let south = [18.0, 19.5, 17.5, 20.5];

let ax = plot::axes()
    .line(day, north).label("north").marker(plot::Marker::Circle)
    .line(day, south).label("south").marker(plot::Marker::Circle)
    .xlabel("day").ylabel("temperature [C]")
    .title("Daily temperature");

let path = env::temp_dir() / f"weather_{process::id()}.svg";
try ax.save(path.as_str());
let svg = try fs::read_to_str(path);
println(f"{svg.starts_with("<svg")} {svg.contains("Daily temperature")}");
try fs::remove_file(path);
```

```output
true true
```

Jupyter 노트북(`nabi jupyter`)에서는 마지막 값이 그래프인 셀이 셀 아래에 SVG를 보여 주므로 파일을 쓸 필요가 없습니다.

## 다음으로

- [밀집 행렬과 희소 행렬의 선형대수](02-linear-algebra.html)는 연립방정식을 풀고 최소제곱으로 맞춥니다.
- [통계](03-statistics.html)는 더 나아가 분포, 여러 검정, 회귀를 다룹니다.
- [packages.nabi-lang.org](https://packages.nabi-lang.org/packages/table)의 `table` API에는 조인, 피벗, 윈도 함수, Parquet, 지연 쿼리가 있습니다.
