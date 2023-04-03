# Tìm hiểu về Learn to Rank Elasticsearch


Học cách xếp hạng (LTR) là sự kết hợp của các kỹ thuật được giám sát và bán giám sát để dự đoán mức độ liên quan của sản phẩm.
Dưới đây là các bước để triển khai một mô hình LTR Elasticsearch nhằm nâng cao mức độ liên quan trong bài toán tìm kiếm cho cửa hàng thương mại điện tử.
Có một vài thuật ngữ mà mình sẽ giữ nó ở tiếng anh, vì dịch ra tiếng Việt nó khá kỳ cục và vô nghĩa :D
<!--more-->

# Các bước thực hiện để triển khai LTR
- Xây dựng Judgment List
- Xây dựng bộ tính năng
- Logging điểm tính năng
- Khởi tạo bộ training dataset
- Lựa chọn thuật toán để build model
- Triển khai model


Ví dụ ta có dữ liệu từ hệ thống analytics như sau:
| Query | Product | Clicks | Add to cart | Purchase |
| --- | --- | --- | --- | --- |
| rambo | A | a1 ~ a1’ | a2 ~ a2’ | a3 ~ a3’ |
| rambo | B | b1 ~ b1’ | b2 ~ b2’ | b3 ~ b3’ |
| rambo | C | c1 ~ c1’ | c2 ~ c2’ | c3 ~ c3’ |
| rambo | D | d1 ~ d1’ | d2 ~ d2’ | d3 ~ d3’ |
| rambo | E | 0 | 0 | 0 |

**Ghi chú**: `a1` là số lần click lên sản phẩm `A`, `a1'` là số điển tương ứng với số lượt click đó, tương tự như vậy cho `a2` và `a3`.

## Xây dựng Judgment List
```text
# qid:1: rambo
# qid:2: ...

4   qid:1   1:a1'   2:a2'  3:a3' # A
3   qid:1   1:b1'   2:b2'  3:b3' # B
3   qid:1   1:c1'   2:c2'  3:c3' # C
2   qid:1   1:d1'   2:d2'  3:d3' # D
0   qid:1   1:0     2:0    3:0   # E
```

Danh sách đánh giá **Ranklib** có định dạng khá chuẩn.
Cột đầu tiên chứa judgment (0-4) cho một sản phẩm (document).
Cột tiếp theo id của cụm từ truy vấn, chẳng hạn như `qid:1`.
Các cột tiếp theo chứa các giá trị của các tính năng được liên kết với cặp tài liệu truy vấn đó.
Trong đó, vế trái là số thứ của tính năng, vế phải là giá trị điểm số của tính năng đó.

`# A` là định danh của sản phẩm (document) cho judgment này.

## Xây dựng bộ tính năng
Một tính năng tương ứng với một trường của sản phẩm thể hiện mức độ liên quan đến sản phẩm đó, ví dụ: `title`, `overview`, `number of views` ..v.v

```json
POST _ltr/_featureset/FEATURE_NAME
{
  "validation": {
    "index": "INDEX_NAME",
    "params": {
      "keywords": "rambo"
    }
  },
  "featureset": {
    "features": [
      {
        "name": "title_bm25",
        "params": [
          "keywords"
        ],
        "template": {
          "match": {
            "title": "{{keywords}}"
          }
        }
      },
      {
        "name": "overview_bm25",
        "params": [
          "keywords"
        ],
        "template": {
          "match": {
            "overview": "{{keywords}}"
          }
        }
      }
    ]
  }
}
```

## Logging điểm tính năng
Điểm tính năng là điểm phù hợp được tính bởi thuật toán BM-25 cho từng tính năng.
Để training model LTR, bạn cần ghi lại các giá trị tính năng.
Đây là một thành phần chính của mô hình LTR: khi người dùng tìm kiếm, chúng ta ghi lại các giá trị tính năng từ bộ tính năng của mình để sau đó chúng ta có thể training model và đánh giá các mô hình hoạt động tốt hay không và dự đoán mức độ phù hợp với tập hợp các tính năng đó.

```json
POST INDEX_NAME/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "terms": {
            "_id": [
              "A",
              "B",
              "C",
              "D"
            ]
          }
        },
        {
          "sltr": {
            "_name": "logged_featureset",
            "featureset": "FEATURE_NAME",
            "params": {
              "keywords": "rambo"
            }
          }
        }
      ]
    }
  },
  "ext": {
    "ltr_log": {
      "log_specs": {
        "name": "log_entry1",
        "named_query": "logged_featureset"
      }
    }
  }
}
```

Sau khi chạy câu truy vấn phía trên, bạn sẽ nhận được kết quả trông như bên dưới.

```json
{
  "hits": {
    "hits": [
      {
        "_id": "A",
        "fields": {
          "_ltrlog": [
            {
              "log_entry1": [
                {
                  "name": "title_bm25"
                },
                {
                  "name": "overview_bm25",
                  "value": 10.558305
                }
              ]
            }
          ]
        },
        "matched_queries": [
          "logged_featureset"
        ]
      },
    ]
  }
}
```

## Khởi tạo bộ training dataset
Bây giờ bạn có thể kết hợp judgment list ban đầu của mình với các feature score để tạo một bộ training dataset.
Đối với dòng tương ứng với sản phẩm (document) `A` cho từ khóa `Rambo`, bây giờ chúng ta có thể thêm như sau:

```text
4   qid:1   1:a1'   2:a2'  3:a3' 4:10.558305 # A
```

## Lựa chọn thuật toán để build model
Bước tiếp theo là lựa chọn thuật toán để xây dựng model. Hiện tại có hai model phổ biến là **ranklib** và **xgboost**.
Các lệnh để build model bạn có thể tham khảo thêm ở trang [Uploading A Trained Model](https://elasticsearch-learning-to-rank.readthedocs.io/en/latest/training-models.html) của LTR


## Triển khai model
Sau khi chạy lệnh build model bạn sẽ nhận được một tệp tin của model, hãy upload nó lên Elasticsearch và trải nghiệm.

## Cách tìm kiếm với LTR
Sử dụng `rescore` kết hợp với model mà bạn đã tạo ở các bước trên vào câu truy vấn của bạn.
Về bản chất kết quả tìm kiếm sẽ được quyết định bởi truy vấn `multi_match`, còn `rescore` có thể hiểu là lần truy vấn thứ hai nhằm thay đổi mức độ ưu tiên của các kết quả trả về.
```json
GET index-name/_search
{
  "_source": {
    "includes": ["title", "overview"]
  },
  "query": {
    "multi_match": {
      "query": "rambo",
      "type": "most_fields",
      "fields": ["title", "overview"]
    }
  },
  "rescore": {
    "window_size": 1000,
    "query": {
      "rescore_query": {
        "sltr": {
          "params": {
            "keywords": "rambo"
          },
          "model": "my_ranklib_model"
        }
      }
    }
  }
}
```

**Lưu ý:** Nếu bạn có một chỉ mục cập nhật thường xuyên, các xu hướng đúng hôm nay có thể không đúng vào ngày mai!
Trên một cửa hàng thương mại điện tử, dép có thể rất phổ biến vào mùa hè, nhưng không thể tìm thấy vào mùa đông.
Các tính năng thúc đẩy mua hàng trong một khoảng thời gian, có thể không đúng với một khoảng thời gian khác.
Bạn nên thường xuyên theo dõi hiệu suất của mô hình và training lại nếu cần.

**Phuc Nguyen**

