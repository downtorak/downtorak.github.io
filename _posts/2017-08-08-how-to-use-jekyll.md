---
layout: post
title:  "Jekyll 사용법"
date:   2017-08-08 15:00:00 +0900
categories: Jekyll
---

* content
{:toc}

### Jekyll
`Jekyll`은 주로 블로그를 만드는데 사용되는 **사이트 생성기**이자 **사이트 테스트기**다.
Jekyll에서 하나의 페이지에 해당하는 Post는 `Markdown`언어를 통해 작성된다. 그래서 `Markdown`은 좀 알아야 써먹을 수 있다.
그 외에 HTML, CSS는 잘 몰라도 테마를 적용해서 사이트 스타일을 쉽게 바꿀 수 있다.
즉, 사이트 리뉴얼한다고 모든 페이지를 바꾸는게 아니라 테마를 적용하고 `Markdown`으로 작성한 Post만 옮겨 넣으면 된다.
그래서 잘만 쓰면 이쁘고 관리도 편한 블로그를 만들 수 있다.

<br>

### 설치하기
Ubuntu 16.04 환경에서 시작.

`Jekyll`은 `gem`으로 제공되기 때문에 `ruby`부터 설치한다.
```no-highlight
$ sudo apt-get install ruby
```

`Jekyll`을 설치할때 `ruby`의 헤더파일을 찾으므로 같이 설치해야 한다.
```no-highlight
$ sudo apt-get install ruby-dev
```

이제 `gem`으로 간단하게 설치.
```no-highlight
$ sudo gem install jekyll
```

이러고 그냥 쓸라하면 `bundler`가 없다며 Dependency 에러가 발생한다. 설치해주자. 에러 안나면 지나가자.
```no-highlight
$ sudo gem install bundler
```

<br>

### 테스트 해보기
`Jekyll`로 사이트를 생성할 적당한 폴더를 마련하자. 나는 `/home/minwoo/workroom/blog/` 여기로 정함.
폴더에 들어가서 `test`라는 사이트를 만들어보자.
```no-highlight
$ jekyll new test
```

실행하면 밑에 처럼 다른 `gem`이 필요하다고 깔아야겠다고 `sudo` 비번을 알려달라고 한다. 알려주자.
```no-highlight
Your user account isn't allowed to install to the system RubyGems.
  You can cancel this installation and run:

      bundle install --path vendor/bundle

  to install the gems into ./vendor/bundle/, or you can enter your password
  and install the bundled gems to RubyGems using sudo.

  Password: 
```

설치 잘하고 사이트 생성에 성공하면 다음과 같은 로그가 나온다.
```no-highlight
New jekyll site installed in /home/minwoo/workroom/blog/test.
```

이제 생성한 사이트를 서버로 돌려보자. 생성한 사이트 폴더로 들어가서 `jekyll serve`를 실행하면 끝.
```no-highlight
$ cd test
$ jekyll serve
```

그럼 아래와 같이 `http://127.0.0.1:4000/`으로 서버를 돌리고 있다고 알려준다.
```no-highlight
Configuration file: /home/minwoo/workroom/blog/test/_config.yml
            Source: /home/minwoo/workroom/blog/test
       Destination: /home/minwoo/workroom/blog/test/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.641 seconds.
 Auto-regeneration: enabled for '/home/minwoo/workroom/blog/test'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

이제 브라우저로 알려준 주소를 접속해보면

![welcome-to-jekyll]({{ site.url }}/assets/2017-08-08-how-to-use-jekyll/01.welcome-to-jekyll.jpg)

환영받는다.

참고로 `127.0.0.1`은 로컬 주소로 원격에서 접속이 안될 수 있다. 그래서 주소를 바꿔주면 원격에서도 접속이 가능하다.
예를 들어 `Jekyll`을 돌리고 있는 서버의 IP주소가 `192.168.56.101`이라면 `-H`옵션과 함께 `jekyll serve`를 실행하면 끝.
```no-highlight
$ jekyll serve -H 192.168.56.101
Configuration file: /home/minwoo/workroom/blog/test/_config.yml
            Source: /home/minwoo/workroom/blog/test
       Destination: /home/minwoo/workroom/blog/test/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
                    done in 0.34 seconds.
 Auto-regeneration: enabled for '/home/minwoo/workroom/blog/test'
    Server address: http://192.168.56.101:4000/
  Server running... press ctrl-c to stop.
```

이제 원격에서 `http://192.168.56.101:4000/`로 접속이 가능하다.

<br>

### 테마 적용하기
테마 사이트에서 원하는 테마를 다운로드한다.
- [http://jekyllthemes.org/](http://jekyllthemes.org/)
- [http://themes.jekyllrc.org/](http://themes.jekyllrc.org/)
- [https://jekyllthemes.io/](https://jekyllthemes.io/)

다운로드한 테마파일을 열어보면 테스트로 만들어본 `test` 사이트의 폴더와 유사한 것을 확인할 수 있다.
그렇다. 테마라는 것이 사이트 폴더 자체다.
`test` 사이트의 폴더 안을 모두 지운 후 테마파일의 압축을 풀어 넣고 다시 브라우저로 접속해보면 테마가 적용된 사이트를 볼 수 있다.

<br>

참고로 테마에 따라 특정한 `gem`을 필요로하는 경우가 있다. 필요한 `gem`이 없을 경우, `jekyll serve` 실행할 때 다음과 같이 뜬다.
```no-highlight
$ jekyll serve -H 192.168.56.101
Configuration file: /home/minwoo/workroom/blog/test/_config.yml
       Deprecation: The 'gems' configuration option has been renamed to 'plugins'. Please update your config file accordingly.
  Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
jekyll 3.5.1 | Error:  jekyll-paginate
```

위의 경우는 `jekyll-paginate`가 없다는 것. 깔아주자.
```no-highlight
$ sudo gem install jekyll-paginate
```

<br>

### Github 연동하기
Github 계정 생성은 알아서...

Github는 `(username).github.io`라는 이름으로 생성한 Repository에 대해 `https://(username).github.io/`의 주소로 호스팅해준다.
즉, `Jekyll`로 생성한 사이트 폴더를 `(username).github.io` Repository에 업로드하면 `https://(username).github.io/`의 주소로 어디서든 접속할 수 있다.

Repository 생성도 알아서...

아까 만든 `test`를 Repository에 업로드한다면 다음과 같이 하면된다.
```no-highlight
$ git clone https://github.com/(username)/(username).github.io.git
$ cp -a ../test/. .
$ git add .
$ git commit -m "(Commit Message)"
$ git push
```

이제 `https://(username).github.io/`로 접속해서 `test`에 생성한 사이트가 나오면 성공.

<br>

### Post 작성하기
앞서 언급한대로 Post는 `Markdown`으로 작성하고 `_posts`폴더에 저장한다. 그 외 Post를 작성하는 규칙은 정해진 파일명을 사용해야한다는 것, YAML 머리말을 포함해야 한다는 것이다.

파일명은 다음 형식을 따라야한다.
```no-highlight
YYYY-MM-DD-title.md
```

예를 들면 이렇게
```no-highlight
2017-08-08-how-to-use-jekyll.md
```

YAML 머리말의 예는 다음과 같다.
```
---
layout: post
title: "Jekyll 사용법"
---
```

Post 작성법은 대체로 동일하지만 적용한 테마에 따라 조금씩 차이가 있을 수 있다. 테마 적용 후 `_posts`폴더에 있는 예제 Post를 참고하도록 하자.