---
layout: blog
title: Apache Parquet 분석
author: Dooyoung Hwang
published: true
category: tech-review
excerpt: |
  Parquet은 Disk에 저장할 목적의 Columnar Format입니다. Parquet는 Data Type별로 적합한 방식으로 Data를 Encoding해 효율적으로 Columnar Data를 저장합니다. 뿐만 아니라, Parquet Data Format은 SQL쿼리 엔진이 Filter Pushdown과 Projection Pushdown을 이용해 쿼리 성능을 최적화할 수 있게 설계되어 있습니다. Parqeut GitHub과 Parquet관련 세미나에서 나온 내용들을 바탕으로 Parquet File 구조, Data Encoding 및 쿼리 최적화 방식에 대해서 정리 해보았습니다.
---

## Parquet vs. Arrow

- Parquet : Disk에 저장할 목적의 columnar format
  - 주로 Storage에 파일형태로 저장
  - I/O reduction에 Priority
    - Projection pushdown : Column pruning
    - Filter pushdown : statistics 기반의 Filtering
  - 대부분 Streaming형태의 Access
- Arrow : In-memory용 columnar format
  - 주로 하나의 Query를 처리하는데 사용되는 format
  - CPU throughput에 Priority
  - Streaming Access + Random Access

## Parquet File Layout

![ApacheParquetReview-Img](/assets/blog/2020/04/ApacheParquetReview.png)

- Hierarchy : Row group > Column chunk > Page > Header + Data (Repetition & Definition Level, non-null values)

- 병렬화 단위 (Unit of parallelization)
  - MapReduce - File/Row Group
  - IO - Column chunk
  - Encoding/Compression - Page

- 파일 구조

    [https://github.com/apache/parquet-format/blob/54e6133e887a6ea90501ddd72fff5312b7038a7c/src/main/thrift/parquet.thrift](https://github.com/apache/parquet-format/blob/54e6133e887a6ea90501ddd72fff5312b7038a7c/src/main/thrift/parquet.thrift)

![ApacheParquetReview-Img1](/assets/blog/2020/04/ApacheParquetReview-1.png)

- Footer : FileMetaData
  - Schema, Version
  - Row groups Metadata : Column Chunk location
- Row group
  - Horizontal Partitioning of data into rows
- Columns Chunks
  - 연속된 Page들로 구성됨
- Page
  - 가장 작은 저장 단위
    - Repetition levels data : List, Set 같은 자료형을 지원하기 위한 데이터
    - Definition levels data : column value 각각의 null, non null 값을 표시
    - Encoded values : non-null column values을 encoding한 데이터
  - Compression/ Encoding 단위 → Page별로 다른 Encoding Type을 쓸 수 있다
  - Supported Encoding : [https://github.com/apache/parquet-format/blob/master/Encodings.md](https://github.com/apache/parquet-format/blob/master/Encodings.md)
    - Plain Encoding
    - Dictionary Encoding
      - The dictionary will be stored in a dictionary page per column chunk.
        - Dictionary page + Data Page
      - If the dictionary grows too big, whether in size or number of distinct values, the encoding will fall back to the plain encoding.
      - Dictionary page format: the entries in the dictionary - in dictionary order - using the plain encoding.
      - Data page format: the bit width used to encode the entry ids stored as 1 byte (max bit width = 32), followed by the values encoded using RLE/Bit-Packing Hybrid. (with the given bit width).
    - Run Length Encoding / Bit-Packing Hybrid
```
            rle-bit-packed-hybrid: <length> <encoded-data>
            length := length of the <encoded-data> in bytes stored as 4 bytes little endian (unsigned int32)
            encoded-data := <run>*
            run := <bit-packed-run> | <rle-run>
            bit-packed-run := <bit-packed-header> <bit-packed-values>
            rle-run := <rle-header> <repeated-value>
```
    - The bit-packing
```
            dec value: 0   1   2   3   4   5   6   7
            bit value: 000 001 010 011 100 101 110 111
            bit label: ABC DEF GHI JKL MNO PQR STU VWX

            bit value: 10001000 11000110 11111010
            bit label: HIDEFABC RMNOJKLG VWXSTUPQ
```
      - bit-packed-run-len and rle-run-len must be in the range [1, 2^31 - 1].
        This means that a Parquet implementation can always store the run length in a signed 32-bit integer.
    - etc : Delta Encoding (INT32, INT64), Delta Length Byte Array(Byte Array), Delta String(Byte Array), Byte Stream Split(Float, Double)
    - Source Code : [https://github.com/apache/parquet-mr/tree/master/parquet-column/src/main/java/org/apache/parquet/column/values](https://github.com/apache/parquet-mr/tree/master/parquet-column/src/main/java/org/apache/parquet/column/values)

- Recommended Configuration
  - Row group size : 512MB - 1GB (larger sequential IO)
  - Data page size : 8KB

## Parquet Schema

- Parquet stores nested data structures in a flat columnar format : Dremel paper from Google
- Schema : describes data structures, Similar to "Protocol buffer(protobuf)"
  - group fields : Nesting
  - repeated fields : Repetition, List나 Set을 표현
  - required, optional : nullable을 표현
```
    message AddressBook {
      required string owner;
      repeated string ownerPhoneNumbers;
      repeated group contacts {
        required string name;
        optional string phoneNumber;
      }
    }
```

  ![ApacheParquetReview-Img2.png](/assets/blog/2020/04/ApacheParquetReview-2.png)

  ![ApacheParquetReview-Img3.png](/assets/blog/2020/04/ApacheParquetReview-3.png)

- Definition levels
  - Nested schema : field가 처음 null이되는 level을 저장 (from root (0 level) to maximum level of the column)
```
    message ExampleDefinitionLevel {
      optional group a {
        optional group b {
          optional string c;
        }
      }
    }
```
    ![ApacheParquetReview-Img4.png](/assets/blog/2020/04/ApacheParquetReview-4.png)
    ![ApacheParquetReview-Img5.png](/assets/blog/2020/04/ApacheParquetReview-5.png)
  - Flat schema : Optional field를 null여부에 따라 0, 1 bit로 encoding
- Repetition levels
  - 새로운 List가 시작할 때 저장되는 값
  - 어느 Level에서 새로운 List를 생성해야 하는 지 Marking하는 정보
```
        message nestedLists {
        	repeated group level1 {
        		repeated string level2;
        	}
        }
```
    ![ApacheParquetReview-Img6.png](/assets/blog/2020/04/ApacheParquetReview-6.png)
    ![ApacheParquetReview-Img7.png](/assets/blog/2020/04/ApacheParquetReview-7.png)
- Flat schema with all fields required : the repetition levels and definition levels are omitted completely

## Parquet Predicate Pushdown

- Page의 Header에는 Statics정보가 Optional하게 들어가 있고 Predicate Pushdown 후 Filter에 부합하지 않는 Page들을 걸러내는 용도로 사용함
```
    /** Data page header */
    struct DataPageHeader {
      /** Number of values, including NULLs, in this data page. **/
      1: required i32 num_values

      /** Encoding used for this data page **/
      2: required Encoding encoding

      /** Encoding used for definition levels **/
      3: required Encoding definition_level_encoding;

      /** Encoding used for repetition levels **/
      4: required Encoding repetition_level_encoding;

      /** Optional statistics for the data in this page**/
      5: optional Statistics statistics;
    }

    /**
     * Statistics per row group and per page
     * All fields are optional.
     */
    struct Statistics {
       /**
        * DEPRECATED: min and max value of the column. Use min_value and max_value.
        *
        * Values are encoded using PLAIN encoding, except that variable-length byte
        * arrays do not include a length prefix.
        *
        * These fields encode min and max values determined by signed comparison
        * only. New files should use the correct order for a column's logical type
        * and store the values in the min_value and max_value fields.
        *
        * To support older readers, these may be set when the column order is
        * signed.
        */
       1: optional binary max;
       2: optional binary min;
       /** count of null value in the column */
       3: optional i64 null_count;
       /** count of distinct values occurring */
       4: optional i64 distinct_count;
       /**
        * Min and max values for the column, determined by its ColumnOrder.
        *
        * Values are encoded using PLAIN encoding, except that variable-length byte
        * arrays do not include a length prefix.
        */
       5: optional binary max_value;
       6: optional binary min_value;
    }
```

- Statics정보를 활용하지 않은 Naive한 구현

  ![ApacheParquetReview-Img8.png](/assets/blog/2020/04/ApacheParquetReview-8.png)

- Statics정보를 활용한 최적화

  ![ApacheParquetReview-Img9.png](/assets/blog/2020/04/ApacheParquetReview-9.png)

(1) Filter Column의  Column Chunk load

(2) Page 별 MIN, MAX Stat 기반으로 Decoding할 Page 선택

(3) Values를 decoding한 후 Filtering으로 Delta vector 도출 (Skipping할 Index)

(4) Column 별 Page Offset Range가 다르므로(Data Size기준으로 Page를 분할하기 때문), B,C,D Column Chunk는 (2)에서 선택된 Delta Vector의 Index와 겹치는 Page들만 Decoding

(5) (3)의 Delta Vector를 B,C,D Page에도 적용해 Filtering

## Speeding Up SELECT Query with Page Indexes

- 질의에 사용되는 Column의 Data를 다른 형태로 적재 후 성능 비교
  - SORTED: generated random data and then sorted the whole data.
  - CLUSTERED_n: generated random data, split it into n-buckets and then sorted each bucket separately. This simulates partially sorted data that occurs in real-life workloads. For example, data sets comprised of timestamps of events.
  - RANDOM: We generated random data and did not apply any sorting.

  ![ApacheParquetReview-Img10.png](/assets/blog/2020/04/ApacheParquetReview-10.png)

- Page Size 조절
  - parquet.page.size
  - parquet.page.row.count.limit : 중복이 많은 Column의 경우 Encoding 후 Data Size가 작아서 전체 Data가 한 Page에 들어갈 수 있으므로 page의 row count 제한도 걸 수 있음

## References

- [https://github.com/apache/parquet-format](https://github.com/apache/parquet-format)
- [https://github.com/apache/parquet-format/blob/master/Encodings.md](https://github.com/apache/parquet-format/blob/master/Encodings.md)
- [https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html)
- [https://www.dremio.com/webinars/columnar-roadmap-apache-parquet-and-arrow/](https://www.dremio.com/webinars/columnar-roadmap-apache-parquet-and-arrow/)
- [https://www.slideshare.net/Hadoop_Summit/the-columnar-roadmap-apache-parquet-and-apache-arrow-102997214](https://www.slideshare.net/Hadoop_Summit/the-columnar-roadmap-apache-parquet-and-apache-arrow-102997214)
- [https://blog.cloudera.com/speeding-up-select-queries-with-parquet-page-indexes/](https://blog.cloudera.com/speeding-up-select-queries-with-parquet-page-indexes/)
