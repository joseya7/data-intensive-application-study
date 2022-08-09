#10. 일괄 처리(Batch processing)
* 세 가지 시스템
    * Services(Online systems)
        * 서비스는 클라이언트로부터 요청이나 지시가 올 때까지 기다린다.
        * 요청 하나가 들어오면 서비스는 가능한 빨리 요청을 처리해서 응답을 되돌려 보내야 한다.
        * **응답 시간**(Response time), **가용성**(Availability) 서비스 성능을 측정할 때 중요한 지표.
    * Batch processing systems (Offline systems)
        * 일괄 처리(Batch processing)은 매우 큰 입력 데이터를 받아 처리하는 작업을 수행.
        * 수 분, 수 일이 걸리기 때문에 사용자가 작업이 끝날 때까지 대기하지 않는다.
        * 주요 성능 지표로는 **처리량**(Throughput)이 대표적.
    * Stream processing systems (near-real-time systems)
        * 스트림 처리(Stream processing)는 일괄 처리 시스템과 마찬가지로 요청에 대해 응답하지 않음.
        * 일괄 처리 작업이 정해진 크기의 입력 데이터를 대상으로 작동한다면, 스트림처리는 입력 이벤트가 발생한 직후 바로 작동.
        * 이런 차이 때문에 스트림 처리 시스템은 같은 작업을 하는 일괄 처리시스템보다 지연 시간이 낮음.

* 맵리듀스와 일괄 처리 시스템
    * *Batch Processing*은 신뢰할 수 있고(**Reliable**), 확장 가능하며(**Scalable**), 유지 보수하기 쉬운(**maintainable**) 애플리케이션을 구축하는데 매우 중요한 구성요소.
    * *Batch Processing Algorithm*인 *MapReduce*는 "구글을 대규모로 확장 가능하게 만든 알고리즘"으로 불림.
    * 현재는 *MapReduce*의 중요성이 떨어지고 있지만, 왜 일괄 처리가 유용한지 알려줌.
    * 이번 장에서는, *MapReduce*를 알아보고, *MapReduce*와 다른 *Batch Processing Algorithm*과 *Framework*를 알아봄.

## 유닉스 도구로 일괄 처리하기 (Batch processing with Unix Tools)
* 먼저, 표준 유닉스가 추구하는 철학은 현대 Batch Processing Algorithm과 비슷한 부분이 있다. 유닉스 철학과 유닉스 도구가 주는 교훈을 통해, Batch Processing을 더 잘 이해할 수 있다.
* 예시를 통해, 표준 유닉스 도구를 사용해 데이터를 처리하는 방법을 살펴보자.
```
216.58.210.78 - - [27/Feb/2015:17:55:11 +0000] "GET /css/typography.css HTTP/1.1"
200 3377 "http://martin.kleppmann.com/" "Mozilla/5.0 (Macintosh; Intel MAC OS X
10_9_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/40.0.2214.115
Safari/537.36"
```
이 로그를 해석해보자면,
* 2015년 2월 27일 UTC 17시 55분 11초에 로그 생성
* 서버가 클라이언트 IP주소 `216.58.210.78`로부터 `css/typography.css`파일에 대한 요청을 받음.
* 비인증 사용자여서 `$remoute_user`가 하이픈으로 표시.
* 응답 상태는 `200`으로 요청은 성공, 응답크기는 `3.377바이트`
* 웹 브라우저는 Chrome, 파일이 `http://martin.kleppmann.com/`이라는 URL에서 참조.

### 단순 로그 분석
* 위와 같은 로그를 표준 유닉스 도구를 통해 "웹 사이트에서 가장 인기가 높은 페이지 5개"를 뽑는 다면
```
cat /var/log/nginx/access.log |   #1
    awk '{print $7}'   |          #2
    sort               |          #2
    uniq -c            |          #3
    sort -r -n         |          #4
    head -n 5          |          #6
```
* \#1. `cat /var/log/nginx/access.log` : 로그를 읽어들인다.
* \#2. `awk '{print $7}'` :`/css/typgraphy.css/` 를 출력
* \#3. `sort` : 요청 URL을 알파벳 순으로 정렬. 
* \#4. `uniq -c`:  중복 제거 및 중복 횟수를 함께 출력
* \#5. `sort -r -n` : 요청 URL을 요청 수 기준으로 다시 정렬.
* \#6. `head` : 맨 앞 5줄만 출력
* 위의 유닉스 쉘의 결과는 아래와 같이 나올 수 있다.
    ```
    4189 /favicon.ico
    3631 /2013/05/25/improving-security-of-ssh-private-keys.html
    2124 /2012/12/05/schema-evolution-in-avro-protocol-buffers-thrift.html
    1309 /
    915  /css/typography.css
    ```
#### 연쇄 명령 대 맞춤형 프로그램(chain of commands vs. custom program)
*  위의 유닉스 명령어들을 루비로 작성할 수 있다.
    ```
    counts = Hash.new(0)                                                #1                   

    File.open('/var/log/nginx/access.log') do |file|
        file.each do |line|
            url = line.split[6]                                         #2
            counts[url] += 1                                            #3
        end
    end

    top5 = coutns.map{|url, count| [count, url]}.sort.reserve[0...5]    #4
    top5.each{|count, url| puts "#{count} #{url}" }                     #5
    ```

* \#1. `counts = Hash.new(0)`  : 각 URL이 몇 번 나왔는지 저장할 해시 테이블이다. 
* \#2. `url = line.split[6] ` : 공백으로 분리, URL이 있는 6번째 인덱스 추출
* \#3. `counts[url] += 1 ` : URL의 카운트를 증가
* \#4. `coutns.map{|url, count| [count, url]}.sort.reserve[0...5]`: 해시 테이블을 내림차순 정렬
* \#5. `top5.each{|count, url| puts "#{count} #{url}" }  ` : 상위 5개 항목을 출력
* 위 두 (유닉스 vs 루비)의 결과는 차이가 없지만, 가장 큰 다른 차이점이 있다 : *실행 흐름*(execution flow). (대용량 파일에서 확연한 차이.)


#### 정렬 대 인메모리 집계
* 위 두 스크립트의 차이는 아래와 같다.
    * **Ruby** : URL 해시 테이블을 메모리에 유지. URL의 수를 맵핑
    * **Unix Tools** : 해시 테이블이 없음. 정렬된 목록에서 같은 URL이 반복해서 나타남.
* *Ruby*, *Unix* 둘 중 무엇이 좋을지는 **서로 다른** URL들이 얼마나 되느냐에 따라 다르다.
    * 백만 개의 로그 항목이 모두 같은 URL 하나만을 가리키면 해시테이블이 유용.
        * 한 개의 URL과 카운트 값을 저장할 만큼만 필요하기 때문.
        * 작업 세트가 작다면 인메모리 해시 테이블도 잘 작동.
* 반면, **허용 메모리**보다 **작업세트가 크다면** 정렬 접근법(*Unix tools*)을 사용하는것이 좋음.
![image](https://user-images.githubusercontent.com/30207544/183413964-6d95486b-59c8-46f9-9ea7-1014f510edf1.png)
    * 이런 정렬 접근법은 디스크를 효율적으로 사용.
        * 기본적으로, SS테이블, LSM트리의 원리와 매우 비슷(p.76)
            1. 데이터 Chunk를 메모리에서 정렬
            2. Chunk를 Segment 파일로 디스크에 저장.
            3. 정렬된 Segment 파일 여러개를 한 개의 큰 정렬 파일로 병합.
        * 이 패턴은 디스크 상에서 좋은 성능을 낼 수 있음.
    * GNU Coreutils(리눅스)에 포함된 `sort` 유틸리티가 위와 비슷.
        * 메모리보다 큰 데이터셋을 자동으로 디스크로 보냄.
        * 여러 CPU코어에서 자동으로 병렬로 정렬.
        * 이를 통해 Unix Pipe 명령어들이 메모리 부족 없이 손 쉽게 큰 데이터셋으로 확장 가능.


### 유닉스 철학
* 연쇄 명령(**Unix chain command**)을 사용해 쉽게 로그 파일을 분석할 수 있는 것은 유닉스 철학과 관련이 있음.
*  1978년에 기술된 유닉스 철학은 아래와 같다.
    1. 각 프로그램은 **한 가지 일만 하도록 작성하라**.
    2. 모든 프로그램의 출력은 아직 알려지지 않은 **다른 프로그램의 입력으로 쓰일 수 있다**고 생각하라.
    3. 소프트웨어를 빠르게 써볼 수 있게 설계하고 구축하라.
    4. 프로그래밍 작업을 줄이려면 미숙한 도움보단 도구를 사용하라.
* `bash` 같은 유닉스 셸을 사용하면 한 가지만 잘하는 작은 프로그램들(`sort`, `uniq`)을 조합하여 강력한 데이터 처리 작업을 쉽게 **구성**할 수 있다.
    * 이런 프로그램 중 다수는 서로 다른 그룹의 사람들이 만들었지만, 유연한 방식으로 함께 **조합**할 수 있다. 유닉스에 이런 결합성을 가능하게 한 것은 **동일 인터페이스**이다.
#### 동일 인터페이스(A uniform interface)
* 특정 프로그램이 다른 **어떤** 프로그램과도 연결 가능하려면 프로그램 **모두**가 같은 입출력 인터페이스를 사용해야 한다.
* 유닉스의 동일 인터페이스(Uniform interface)는 `File`이다.
* `File`은 정렬된 바이트의 연속이여서 매우 단순하다.
    * 이 같은 특성때문에, 같은 인터페이스로 `Filesystem`, 통신채널(`Unix Socket`, `stdin`, `stdout`), 장치 드라이버(`/dev/audio/`), `TCP connection` 등 여러 가지 것을 표현할 수 있다.
* 완벽하지 않음에도 유닉스의 동일 인터페이스(`Uniform interface`)는 여전히 *대단하다*. 
    * 예를 들어, 이메일 내용을 파이프로 연결해서, 데이터를 파싱하여 스프레드 시트에 넣고, 결과를 웹에 올리는 것은 쉽게 할 수 없다.

    * 오늘날 유닉스 도구처럼 **매끄럽게 협동**하는 프로그램이 있다는 것은 정상이 아니라 예외적이다.

#### 로직과 연결의 분리(Separtion of logic and wiring)
`cat select_user_info.sql | ./access_db.sh main > 'user_info.tsv'`
* 유닉스 도구는 표준 입력(`stdin`)과 표준 출력(`stdout`)을 사용한다.
    * 프로그램을 실행하고 아무것도 설정하지 않는다면 `stdin`은 키보드로부터 들어오고 `stdout`은 화면으로 출력한다.
    * 유닉스는 파이프를 통해 한 프로세스의 `stdout`을 다른 프로세스의 `stdin`과 연결한다.
    * 이처럼, 프로그램에서 입출력을 연결하는 부분을 분리하면 작은 도구로부터 큰 시스템을 구성하기가 훨씬 수월해진다.
#### 투명성과 실험(Transparency and Experimentation)
* 유닉스가 성공적인 이유는 진행사항을 파악하기가 쉽기 때문이다.
    1. 유닉스 명령에 들어가는 입력 파일은 **불변**(immutable)로 처리된다.
    2. 어느 시점이든 파이프라인을 중단하고, `less`를 통해 원하는 형태의 출력이 나오는지 확인할 수 있다.
    3. 특정 파이프라인 단계의 출력을 파일에 쓰고, 그 파일을 다음 단계의 입력으로 사용할 수 있다.
* 이러한 이유 때문에 유닉스 도구는 실험(Experimentation)을 할 때 자주 사용될 수 있다.

## 맵리듀스와 분산 파일시스템
* MapReduce는 Unix도구와 비슷한 면이 있지만, 수천 대의 장비로 분산해서 실행이 가능하다는 점에서 차이가 있다.
* MapReduce와 Unix는 아래와 같은 유사한 면을 가지고 있다.
    * 단일 MapReduce작업은 하나 이상의 입력을 받아, 하나 이상의 출력을 만들어낸다는 점.
    * MapReduce작업은 입력을 수정하지 않기 때문에, 출력을 생산하는 것 외에 다른 부수효과가 없다는 점.
    * MapReduce작업은 분산 파일 시스템상의 파일을 입력과 출력으로 사용한다는 점. 
* Hadoop MapReduce구현에서 이 파일 시스템은 HDFS라고 한다.
* HDFS는 **비공유 원칙**을 기반으로 하여, 공유 디스크 방식과는 반대다.
    * 공유 디스크 방식은 맞춤형 하드웨어를 사용하거나, 특별한 인프라를 사용해야 하지만, 비공유 방식은 일반적인 데이터센터 네트워크에 연결된 컴퓨터면 충분하다.
* HDFS는 매우 큰 하나의 파일시스템이되고, 데몬 프로세스를 통해, HDFS내의 실행 중인 모든 장비의 디스크를 사용할 수 있다.
![image](https://user-images.githubusercontent.com/30207544/183413405-2250da6c-3ff0-4752-a871-354c68706658.png)

    * **데몬 프로세스**는 다른 노드가 해당 장비에 저장된 파일에 접근 가능하게끔 네트워크 서비스를 제공.
    * **NameNode**라고 부르는 중앙 서버는 특정 특정 파일 블록이 어떤 저장됐는지 추적.
* HDFS에서 장비가 죽거나 디스크가 실패하는 경우에 대비하기 위해 파일 블록은 여러 장비에 복제.
* HDFS는 확장성이 뛰어나, 수만 대의 장비를 묶어 실행할 수 있고 수백 페타바이트에 달하는 용량을 가질 수도 있다.
* HDFS를 이용한 데이터 저장과 접근 비용은 범용 하드웨어와 오픈소스 소프트웨어를 사용하기 때문에 동급 용량의 전용 저장소를 사용하는 비용보다 훨씬 저렴하다.
### 맵리듀스 작업 실행하기
* 맵리듀스는 HDFS와 같은 분산 파일 시스템 위에서 대용량 데이터셋을 처리하는 코드를 작성하는 프로그래밍 프레임워크
* 맵리듀스의 데이터 처리패턴은 위의 로그 분석예제와 매우 비슷.
    ```
    로그 분석 예제                               맵리듀스
    cat /var/log/nginx/access.log |   <->   #1. 입력 파일을 읽어, Record로 쪼갠다(\n).
    awk '{print $7}'   |              <->   #2. Record마다 Mapper 함수 호출. 매핑(key:url, value:  )
    sort               |              <->   #3. key를 기준으로 정렬       
    uniq -c            |              <->   #4. key-value에 Reduce 함수 호출. 인접한 레코드의 수를 센다.
    sort -r -n         |              <->   #5.
    head -n 5          |         
    ```
    * 맵리듀스 작업은 4가지 단계로 수행.
        * *1단계*는 파일을 나누어 레코드를 만듬.
        * *3단계*는 정렬단계로 맵리듀스에 내재하된 단계라서 작성할 필요가 없음.
        * *2단계*(Map), *4단계*(Reduce)는 사용자가 직접 작성한 데이터 처리 코드.
* 맵리듀스 작업을 생성하려면 Mapper와 Reducer라는 두 가지 콜백함수를 구현해야한다.
![image](https://user-images.githubusercontent.com/30207544/183419940-57efee4c-5420-4e44-8b4c-f94fe0c6ffeb.png)
    * **Mapper**
        * Mapper는 모든 입력 레코드마다 한 번씩만 호출.
        * Mapper는 Key, Value값을 추출하는하는 작업.
        * Mapper는 정렬에 적합한 형태로 데이터를 준비하는 역할.
    * **Reducer**
        * Reducer는 Mapper가 생성한 Key-Value쌍을 받아 같은 키를 가진 레코드를 모음.
        * Reducer함수는 해당 값의 집합만큼 반복. 
        * Reducer의 예로, 동일한 URL이 출현한 횟수.
        * Reducer는 정렬된 데이터를 가공하는 역할.
* 웹 서버 로그 예제에서, 5번째 단계를 보면 두 번째 정렬 명령어가 있는데, 맵리듀스에서 두 번째 정렬이 필요하다면, 첫 번째 작업의 출력을 두 번째 작업의 입력으로 사용할 수 있다.
#### 맵리듀스의 분산 실행
![image](https://user-images.githubusercontent.com/30207544/183423220-18e997e0-7686-488b-9990-31f0b8d63f9f.png)
* Unix 명령어 파이프라인(command pipeline)과 맵리듀스의 가장 큰 차이점은, 맵리듀스가 병렬로 수행하는 코드를 직접 작성하지 않고도 여러 장비에서 동시에 처리가 가능하다는 점.
* 맵리듀스의 병렬 실행은 파티셔닝을 기반.
* 입력으로 HDFS상의 디렉터리를 사용. 각 입력 파일은 보통 크기가 수백 MB.
* MapTask 코드는 맵리듀스 프레임워크를 통해 장비에 복사. 장비는 입력 파일을 읽기 시작하여 한 번에 레코드 하나씩 읽어 Mapper 콜백함수로 전달.
* Reducer는 전달받은 키의 해시값을 사용하여, 같은 Key를 가진 Key-value Pair들을 같은 Reducer에서 처리하게 된다.
* Reducer를 기준으로 파티셔닝하고 정렬한 뒤 Mapper로부터 데이터 파티션을 복사하는 과정을 **Shuffle**이라고 한다.
* Reduce Task는 Mapper로부터 받은 데이터를 정렬된 순서로 병합한다.
* Reducer는 Key와 Iterator를 인자로 호출하고, 이 Iterator로 전달된 Key와 동일한 Key를 가진 레코드를 모두 훑을 수 있게 됨.
* 출력 레코드는 분산 파일 시스템에 파일로 기록된다.

#### 맵리듀스 워크플로
* 맵리듀스는 Task를 연결해 워크플로(Workflow)를 구성.
    * Task하나의 출력을 다른 MapReduce Task의 입력으로 사용하는 방식.
    * 연결된 MapReduce Task는 유닉스 명령 파이프라인(소량의 메모리 버퍼를 사용해 한 프로세스의 출력이 다른 프로세스의 입력으로 직접 전달되는 방식)과는 조금 다르다.
    * MapReduce Workflow는 각 명령의 출력을 임시 파일에 쓰고, 그 임시 파일로부터 입력을 읽는 방식이다.
* Workflow상에서 해당 Task의 입력 디렉터리를 생성하는 선행작업이 끝나야먄 다음 Task가 시작될 수 있다.
    * Airflow, Oozie, Azkaban, Luigi등이 MapReduce Workflow의 의존성을 관리하는 스케줄러 도구가 될 수 있다.
    * 큰 조직에서는,  50~100개의 Workflow를 사용하는 것이 일반적. 이런 복잡한 데이터 Flow를 관리하기 위해서는 위와 같은 스케줄러 도구가 필요.

## 다음시간주제
*  리듀스 사이드 조인과 그룹화
*  맵 사이드 조인
*  일괄 처리 워크플로의 출력
* 하둡과 분산 데이터베이스의 비교
