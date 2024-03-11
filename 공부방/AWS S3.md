#### Simple Storage Service
---
S3는 어디서나 원하는 양의 데이터를 저장하고 검색할 수 있도록 구축된 객체 스토리지입니다.데이터는 버킷 내에 객체 단위로 저장합니다. 객체는 각각의 데이터, 버킷은 객체의 컨테이너를 뜻합니다.

S3 버킷은 해당 버킷을 생성한 계정이 소유권을 가집니다. 각 계정당 최대 100개의 버킷을 만들 수 있으며 버킷의 개수에 따른 성능상의 차이는 없습니다.

버킷에 저장할 수 있는 객체의 수에는 제한이 없지만 버킷 안에 새로운 버킷을 만들 수는 없습니다.

버킷은 디렉터리의 개념 없이 경로의 역할을 하는 prefix로 구성할 수 있습니다. 각각의 prefix마다 초 당 최소 3,500개의 PUT/POST/DELETE 요청을 받을 수 있고 5,500개의 GET 요청을 받을 수 있습니다. 이러한 요청의 제한은 요청 빈도에 따라서 S3가 오토 스케일링을 수행합니다. 그러나 스케일링은 점진적으로 이루어지기 때문에 스케일링이 이루어지는 동안 503 오류가 발생할 수 있습니다. 만일 발생하는 요청의 수가 이러한 제한을 초과하는 경우 추가적인 prefix를 지정하여 요청을 분산시켜 성능을 개선할 수 있습니다.

#### 참고
---
https://devocean.sk.com/blog/techBoardDetail.do?ID=163606
https://docs.aws.amazon.com/ko_kr/AmazonS3/latest/userguide/optimizing-performance.html
https://stackoverflow.com/questions/52443839/s3-what-exactly-is-a-prefix-and-what-ratelimits-apply
