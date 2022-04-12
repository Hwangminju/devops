# Jenkins 빌드/배포 오류

## 1. git clone 단계에서 오류 발생

#### [1 오류 내용] 

git clone 시에 origin을 cloning 할 수 없다는 오류 발생

![image-20211221151033775](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 에러2.png)

빌드 시에, jenkins-slave에서 git fetch, git clone 등 git을 연동할 때, 위와 같이 Failed to connect 에러 또는 Permission denied 에러가 발생.



#### [1 해결 내용]

안녕하세요. 오픈채널사업TF 박용진입니다.

휴유... 다행히 해결했습니다.

 

jenkins-slave 에서 git fetch 등 git 연동할 때 아래와 같이 Permission denied, HTTP Basic: Access denied 에러가 발생했었는데요, credential에는 문제가 없었습니다.

다만, 두 번째 스크린샷에서 보시는 것처럼 /home/jenkins/ 경로에 .gitconfig 외에 **.git-credentials** **라는** **캐시** **파일이** **있었고****,** **여기에** **엉뚱한** **값이** 들어가 있었습니다.

 

**해당** **파일** **삭제** **후** **정상**적으로 수행할 수 있었습니다.

향후 동일한 에러 발생 시 위와 같이 조치하시면 됩니다.

 

PS. 해당 파일이 여기에 왜 생겼는지는 의문이에요.

 ![](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 에러.png)

 ![](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 해결.png)

감사합니다.



## 2. Deploy to Cluster 단계에서 오류 발생

![image-20211228144729428](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 에러3.png)

#### [2-1. i/o timeout 오류 내용]

1) Deploy to Cluster 단계에서 helm upgrade 시에 i/o timeout 발생 (개발)

2) 오랫동안 배포를 진행하지 않으면 해당 오류 발생

#### [2-1. i/o timeout 오류 내용]

tiller pod 삭제 후 재기동하면 해결



#### [2-2. no deployed releases 오류 내용]

2) Deploy to Cluster 단계에서 helm upgrade 시에 아래 오류 발생

#### [2-2. no deployed release 해결 내용]

```
appdu-demo6-prd-deploy has no deployed releases
```

- helm ls -a로 helm으로 released된 모든 리소스 조회

  ```
  user@RP1015108010:~$ helm ls -a
  NAME                    REVISION        UPDATED                         STATUS  CHART                           APP VERSION   NAMESPACE
  appdu-demo6-prd-deploy  1               Tue Dec 28 16:08:13 2021        DELETED appdu-demo6-prd-deploy-0.1.0    1.0           appdu-demo6
  ```

- release된 helm 리소스(appdu-demo-prd-deploy) 삭제

  ```
  user@RP1015108010:~$ helm delete --purge appdu-demo6-prd-deploy
  release "appdu-demo6-prd-deploy" deleted
  ```

  

## 3. 배포 (Deploy to Cluster) 단계에서 마스터 노드 metrics 파드 연결 오류

![](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 배포 에러(merics pod).PNG)



#### [3-1. 오류 내용]

Deploy to Cluster 단계에서 Could not get apiVersions from Kubernetes: unable to retrieve the complete list of server APIs: metrics.k8s.io/v1beta1 메세지 발생하며 배포 진행 불가

#### [3-1. 해결 내용]

- 현황
  - 지난주에 (13일, 목요일) 인프라 팀에서 마스터노드 (1~3번 노드) CPU 증설작업 시도했으나, Cnode 자원부족으로 노드 증설 진행 불가
  - 14일 금요일에 다시 증설작업 진행 => 완료
  - 증설작업 과정에서 master 노드 재기동되면서 metrics POD가 정상적으로 못 올라옴 => 배포 오류 발생
- 해결 내용
  - 인프라 팀 (Cloud솔루션개발팀 김유민 대리)에 연락해서 마스터 노드의 metrics POD 재기동 요청

 

## 4. jenkins 빌드 시 오류 (jenkins slave 오프라인 상태)

#### [4-1. 오류 내용]

- jenkins slave pod 상태가 모두 오프라인 상태로 빠지면서 "almost complete" 메세지만 유지되고, 진행 불가
- devops 네임스페이스에 있는 **jenkins master pod 재기동 시도 (Arsenal 쪽 네임스페이스), 남아있는 빌드 대기 목록, slave 전부 삭제** => 오류 다시 발생
- Arsenal 쪽에서는 다수의 네임스페이스에서 jenkins job을 여러 번 눌러서 쓰레기 slave 파드가 남았던 경우가 있었음 => 해당 문제는 아니었음



- 고준호 과장님 확인 결과, **appdu02 개발 노드에 대해 그라파나에서 노드 리소스 모니터링 정보가 안 뜨는 문제** 확인 => 해당 노드를 통해서 뜨는 파드들이 영향을 받음.
- 파드들이 못 뜨는데 oc describe node 명령어로 봤을 땐 리소스가 부족한 것 같진 않았으나, 그라파나에서는 리소스 정보가 안 나옴.



- 박지선 대리님 확인 결과, appdu02의 node-exporter만 비정상 상태 확인 => **appdu02 노드 node-exporter** 재기동 시도 => node-exporter 재기동도 정상적으로 올라오지 않음.

- **워커노드에서 docker, origin 서비스 재시작** 시도 => 재시작해도 pod 기동 불가

- 네트워크 쪽이 뭔가 충돌이 나는것 같음 => 워커노드 상태도 NotReady

- ***appdu02 노드 재기동*** 시도 => 빌드 잘 되는 것 확인



#### [4-1. 해결 내용]

이번처럼 젠킨스를 여러 개 돌렸는데 하나의 노드에 많은 파드들이 순간적으로 할당되면 이런 현상(빌드 오류)이 발생했던 적이 있었다고 함 (네트워크 에러 발생)

=> 노드에 많은 파드들이 러닝 중일때는 문제가 아니고 **많은 파드들이 create될 때** 발생하는 것 같음

젠킨스 수행하면서 slave 파드가 뜰 때 jenkins-slave container 3개를 갖고 올라오면서 컨테이너가 갑자기 많아져서 발생하는 것으로 추정 => **노드 재기동**을 통해 해소

- 다음에 비슷한 오류 발생 시에는 박지선 대리님께 문의



#### [4-1. 오류 재발생]

- appdu02 노드 재기동 후 jenkins 다시 stop

- 고준호 과장님 확인해보니 okd콘솔도 이상함 (okd콘솔도 개발 클러스터에 존재)

- 김유민 대리님 확인 결과, **마스터노드 리소스 사용량 이슈** 발생 확인

  => **마스터 노드의 CPU 사용율이 높아서 긴급 증설할 예정**이며 가능 시간 확인 중 (증설 작업 중에 배포 등 불가)

- 마스터 노드는 인프라에서 관리하니까 appdu 영역은 아님. **(쿠버네티스 클러스터를 관리하는 서버)**

- *마스터 노드 용량 때문에 그 밑에 앱두 포함해서 다른 서비스들도 빌드가 안됨 (아스날도 영향을 받음)*



#### 5. Deploy to cluster 단계에서 tiller tcp 오류

helm 2버전에서 자주 발생하는 문제.

tiller 실행한지 오래됬으면 tiller 내렸다가 다시 올리면 됨!

![image-20220120104533811](D:\10151080\2. Appdu\0. 중요사항\4. 이슈 사항\jenkins 배포 에러(tiller tcp).png)



#### 6. backend pod가 Image Pull Back-off status로 지속되는 오류

Image pull secrets:  default-dockercfg-sld9c (not found)
Mountable secrets:   default-token-4bh8t
                     			default-dockercfg-sld9c (not found)



이미지를 못 불러오는 상태