## 天马行情历史数据

创建 mysql 映射数据库

```bash
CREATE DATABASE if not exists hiquotation_min_orig_sz ENGINE = MySQL('192.168.11.192:3306', 'hiquotation_min_orig_sz', 'dev', 'Dev120qfp');
```

创建新库和新表

```bash
# 创建新库
CREATE DATABASE if not exists hiquotation_min_orig_ch;
USE hiquotation_min_orig_ch;
# 创建新表
CREATE TABLE tb_hisbar_min_2020
(
    symbol varchar(32) COMMENT '标的',
    open decimal(18,4) COMMENT '开盘价',
    high decimal(18,4) COMMENT '最高价',
    low decimal(18,4) COMMENT '最低价',
    close decimal(18,4) COMMENT '收盘价',
    volume UInt64 DEFAULT 0 COMMENT '成交量',
    turnover decimal(18,4) DEFAULT 0.0000 COMMENT '成交额',
    interval UInt8 COMMENT '周期数,1分钟为1',
    starttime UInt64 DEFAULT 0 COMMENT '开始时间',
    endtime UInt64 DEFAULT 0 COMMENT '结束时间'
) ENGINE = MergeTree() ORDER BY (symbol,interval,starttime);

```

插入数据

```bash
INSERT INTO tb_hisbar_min_2020(symbol, open, high, low, close, volume, turnover, interval, starttime, endtime) select symbol, open, high, low, close, volume, turnover, interval, starttime, endtime from hiquotation_min_orig_sz.tb_hisbar_min_2020;
```

---

## 未复权行情表

```bash
# 报错，不知道什么原因
CREATE DATABASE if not exists jrtz_hg_sz ENGINE = MySQL('192.168.10.211:3306', 'jrtz_hg', 'tester', 'test0904');
# 创建新库
CREATE DATABASE if not exists jrtz_hg_ch;
USE jrtz_hg_ch;
# 创建新表
CREATE TABLE quotefact 
(
    F2000 varchar(6) COMMENT '股票代码',
    VAR_CL varchar(2) COMMENT '证券类型',
    MKT_CL varchar(1) COMMENT '交易市场代码',
    F2100 DateTime COMMENT '行情时间',
    F2101 Nullable(Decimal64(4)) COMMENT '昨收盘价',
    F2102 Nullable(Decimal64(4)) COMMENT '开盘价',
    F2103 Nullable(Decimal64(4)) COMMENT '最高价',
    F2104 Nullable(Decimal64(4)) COMMENT '最低价',
    F2105 Nullable(Decimal64(4)) COMMENT '最新价',
    F2106 Nullable(Decimal64(4)) COMMENT '成交量',
    F2107 Nullable(Decimal64(4)) COMMENT '成交金额',
    CREAT_TM DateTime COMMENT '创建时间',
    UPDT_TM DateTime COMMENT '更新时间',
    IS_VLD UInt8 COMMENT '是否有效',
    RMRK Nullable(varchar(100))COMMENT '备注'
) ENGINE = MergeTree() ORDER BY (F2000,VAR_CL,MKT_CL,F2100);
# 插入数据
INSERT INTO quotefact(F2000, VAR_CL, MKT_CL, F2100, F2101, F2102, F2103, F2104, F2105, F2106, F2107, CREAT_TM, UPDT_TM, IS_VLD, RMRK) select F2000, VAR_CL, MKT_CL, F2100, toDecimal64OrNull(F2101, 4),
toDecimal64OrNull(F2102, 4),
toDecimal64OrNull(F2103, 4),
toDecimal64OrNull(F2104, 4),
toDecimal64OrNull(F2105, 4),
toDecimal64OrNull(F2106, 4),
toDecimal64OrNull(F2107, 4),
CREAT_TM, UPDT_TM, IS_VLD, RMRK from mysql('192.168.10.211:3306', 'jrtz_hg', 'quotefact', 'tester', 'test0904');
```

---

## 复权行情表

```bash
USE jrtz_hg_ch;
# 创建新表
CREATE TABLE quote 
(
    F2000 varchar(6) COMMENT '股票代码',
    VAR_CL varchar(2) COMMENT '证券类型',
    MKT_CL varchar(1) COMMENT '交易市场代码',
    F2400 DateTime COMMENT '行情时间',
    F2401 Nullable(Decimal64(4)) COMMENT '昨收盘价',
    F2402 Nullable(Decimal64(4)) COMMENT '开盘价',
    F2403 Nullable(Decimal64(4)) COMMENT '最高价',
    F2404 Nullable(Decimal64(4)) COMMENT '最低价',
    F2405 Nullable(Decimal64(4)) COMMENT '最新价',
    F2406 Nullable(Decimal64(4)) COMMENT '成交量',
    F2407 Nullable(Decimal64(4)) COMMENT '成交金额',
    CREAT_TM DateTime COMMENT '创建时间',
    UPDT_TM DateTime COMMENT '更新时间',
    IS_VLD UInt8 COMMENT '是否有效',
    RMRK Nullable(varchar(100))COMMENT '备注'
) ENGINE = MergeTree() ORDER BY (F2000,VAR_CL,MKT_CL,F2400);
# 插入数据
INSERT INTO quote(F2000, VAR_CL, MKT_CL, F2400, F2401, F2402, F2403, F2404, F2405, F2406, F2407, CREAT_TM, UPDT_TM, IS_VLD, RMRK) select F2000, VAR_CL, MKT_CL, F2400, toDecimal64OrNull(F2401, 4),
toDecimal64OrNull(F2402, 4),
toDecimal64OrNull(F2403, 4),
toDecimal64OrNull(F2404, 4),
toDecimal64OrNull(F2405, 4),
toDecimal64OrNull(F2406, 4),
toDecimal64OrNull(F2407, 4),
CREAT_TM, UPDT_TM, IS_VLD, RMRK from mysql('192.168.10.211:3306', 'jrtz_hg', 'quote', 'tester', 'test0904');
```

