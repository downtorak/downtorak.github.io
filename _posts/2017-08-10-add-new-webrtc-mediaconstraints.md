---
layout: post
title:  "WebRTC에 MediaConstraints 추가하기"
date:   2017-08-10 17:00:00 +0900
categories: Jekyll
---

* content
{:toc}

### MediaConstraints 입력 분석
`webrtc/api/mediaconstraintsinterface.h` 파일을 보면 다음의 소스를 찾을 수 있다.
```cpp
class MediaConstraintsInterface {
 public:
  struct Constraint {
    Constraint() {}
    Constraint(const std::string& key, const std::string value)
        : key(key), value(value) {
    }
    std::string key;
    std::string value;
  };

  class Constraints : public std::vector<Constraint> {
   public:
    bool FindFirst(const std::string& key, std::string* value) const;
  };

  virtual const Constraints& GetMandatory() const = 0;
  virtual const Constraints& GetOptional() const = 0;
```

입력받은 모든 Constraint는 지원하든 않든 일단 key와 value로 vector에 저장된다.



