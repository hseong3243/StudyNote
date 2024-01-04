이전 [일대다 페이지네이션 최적화하기](https://hseong.tistory.com/76)에서 애니프렌즈 프로젝트의 코드 일부를 예로 들며 BatchSize와 프로젝션을 이용한 일대다 컬렉션 페이징 쿼리를 최적화하였습니다. 추가로 효율적이지 않은 쿼리가 있나 탐색하던 중 다음 로직을 발견하였습니다.
```java
public record FindAnimalResponse(  
    Long animalId,  
    String animalName,  
    String shelterName,  
    String shelterAddress,  
    String animalImageUrl  
) {  
  
    public static FindAnimalResponse from(Animal animal) {  
        return new FindAnimalResponse(  
            animal.getAnimalId(),  
            animal.getName(),  
            animal.getShelter().getName(),  
            animal.getShelter().getAddress(),  
            animal.getImages().get(0)  
        );  
    }
}
```

```java
@Override  
public Page<Animal> findAnimals(..., Pageable pageable) {
	List<Animal> animals = query.selectFrom(animal)  
	    .join(animal.shelter).fetchJoin()  
	    .leftJoin(animal.shelter.image).fetchJoin()  
	    .where(  
	        ...
	    )  
	    .orderBy(animal.createdAt.desc())  
	    .offset(pageable.getOffset())  
	    .limit(pageable.getPageSize())  
	    .fetch();
    ...
}
```

```java
@Entity
public class Animal {
	...
	@BatchSize(size = 100)  
	@OneToMany(mappedBy = "animal", cascade = CascadeType.PERSIST,  
	    fetch = FetchType.LAZY, orphanRemoval = true)  
	private List<AnimalImage> images = new ArrayList<>();
	...
}
```

보호 동물 서비스는 애니프렌즈 프로젝트에서 가장 전면에 노출되는 기능입니다. 그만큼 보호 동물 목록 조회는 가장 많은 요청이 발생될 것으로 예상되는 API 중 하나입니다. 

로직을 살펴보면 일대다 컬렉션 페이징을 위하여 `BatchSize`를 사용하고 있는 것을 알 수 있습니다. 하지만 조회해온 이미지 중 실제로 사용하는 것은 리스트의 가장 첫번째 요소 뿐입니다. 한 번 조회 시 20마리의 보호 동물을 조회하고 일대다로 연결된 보호 동물 이미지를 in 절로 총 100건을 추가로 조회해오기 때문에 상당히 비효율적입니다.
![[스크린샷 2023-12-14 오후 10.51.32.png]]

포스트맨으로 실제 api 요청을 보내보니 생각한대로 상당히 긴 시간이 걸리는 것을 확인할 수 있었습니다.

## 쿼리 최적화
---
앞서 이야기한대로 20마리의 보호 동물 조회 시 필요한 보호 동물 이미지는 각각 1건씩, 20건만 있으면 됩니다. 그러나 현재 in절로 100건의 이미지를 조회하면서 80건의 이미지는 그대로 버려지고 있습니다.

이를 개선하기 위한 방법으로는 2가지가 생각납니다.

1. `보호 동물` 목록 조회 1회, `보호 동물 이미지` 각각 1건씩만 in절 조회 1회, 애플리케이션 상에서 조인 수행
2. 필요한 데이터만 프로젝션 1회

만약 보호 동물 목록 조회에서 `보호 동물` 엔티티의 데이터 대부분을 그대로 사용하고 있다면 1번만으로 충분할 것 같습니다. 하지만 현재 나가고 있는 쿼리와 실제로 사용하고 있는 데이터를 비교해보니 `select`에 불필요한 데이터가 상당수 포함되어 있음을 알 수 있었습니다.
```sql
select
	a1_0.animal_id,
	a1_0.active,
	a1_0.is_adopted,
	a1_0.birth_date,
	a1_0.breed,
	a1_0.created_at,
	a1_0.gender,
	a1_0.information,
	a1_0.name,
	a1_0.is_neutered,
	s1_0.shelter_id,
	s1_0.address,
	s1_0.address_detail,
	s1_0.is_opened_address,
	s1_0.created_at,
	s1_0.device_token,
	s1_0.email,
	i1_0.shelter_image_id,
	i1_0.created_at,
	i1_0.image_url,
	s1_0.name,
	s1_0.password,
	s1_0.phone_number,
	s1_0.spare_phone_number,
	a1_0.type,
	a1_0.weight 
from
	animal a1_0 
join
	shelter s1_0 
		on s1_0.shelter_id=a1_0.shelter_id 
left join
	shelter_image i1_0 
		on s1_0.shelter_id=i1_0.shelter_id 
order by
	a1_0.created_at desc 
limit
	?,?
```

```java
public record FindAnimalResponse(  
    Long animalId,  
    String animalName,  
    String shelterName,  
    String shelterAddress,  
    String animalImageUrl) {
	...
}
```

`select` 절에 포함된 26건의 컬럼 중 실제로 사용하고 있는 데이터는 5건밖에 되지 않습니다. 이 정도면 프로젝션을 수행했을 때 좀 더 의미있는 개선을 확인할 수 있을 것이라 판단하였습니다.

적당한 DTO를 만들고 프로젝션을 이용해서 개선한 쿼리는 다음과 같습니다.
```java
List<FindAnimalsResult> animals = query  
    .select(new QFindAnimalsResult(  
        animal.animalId,  
        animal.name.name,  
        animal.createdAt,  
        animal.shelter.name.name,  
        animal.shelter.addressInfo.address,  
        ExpressionUtils.as(  
            select(animalImage.imageUrl.max())  
                .from(animalImage)  
                .where(animalImage.animal.eq(animal)) 
            , "animageImageUrl")  
    ))  
    .from(animal)  
    .join(animal.shelter)  
    .leftJoin(animal.shelter.image)  
    .where(  
        animalTypeContains(type),  
        animalActiveContains(active),  
        animalIsNeutered(neuteredFilter),  
        animalAgeContains(age),  
        animalGenderContains(gender),  
        animalSizeContains(size)  
    )  
    .orderBy(animal.createdAt.desc())  
    .offset(pageable.getOffset())  
    .limit(pageable.getPageSize())  
    .fetch();
```

```sql
select
	a1_0.animal_id,
	a1_0.name,
	a1_0.created_at,
	s1_0.name,
	s1_0.address,
	(select
		max(a2_0.image_url) 
	from
		animal_image a2_0 
	where
		a2_0.animal_id=a1_0.animal_id) 
from
	animal a1_0 
join
	shelter s1_0 
		on s1_0.shelter_id=a1_0.shelter_id 
order by
	a1_0.created_at desc 
limit
	?,?
```

`보호 동물`과 `보호소`에서 필요한 데이터 5건만 `select` 절에 포함시켰고 `보호 동물 이미지`는 서브 쿼리로 한 건만 가져오도록 개선하였습니다. 

이전과 동일하게 포스트맨으로 다시 요청을 보냈을 때 프로젝션을 사용하기 전에는 약 1000ms 정도가 소모되던 것에서 약 260ms 정도로 개선된 것을 확인할 수 있습니다.
![[스크린샷 2023-12-14 오후 11.19.06 1.png]]

## 성능 비교
---
포스트맨을 이용해서 비교를 수행하는 경우 요청할 때마다 응답 시간의 차이가 발생하였습니다. 심한 경우에는 동일한 요청에서도 500ms 정도까지 차이가 발생하였습니다.

조금 더 정확한 성능 측정을 위해 ngrinder를 이용하여 개선 전과 개선 후를 비교해보았습니다.
![[스크린샷 2023-12-14 오후 11.28.25.png]]

테스트 환경은 다음과 같습니다.

- AWS EC2 small 1대 + RDS micro 1대
- EC2에는 jar 파일로 배포된 스프링 부트 애플리케이션 외에 로그 수집을 위한 promtail과 nginx가 실행 중
- ngrinder는 별도의 EC2 인스턴스에서 실행
- 데이터는 `보호 동물` 100,000건, 일대다로 연관된 `보호 동물 이미지` 500,000건

가상 유저는 프로그래머스 데브코스의 교육생 전원이 동시에 접속한다고 가정하여 100명, 테스트 시간은 5분으로 설정하였습니다. 

우선 개선 전의 리포트입니다.
![[개선전(1).png]]
![[개선전(2).png]]

평균 TPS(Transaction Per Seconds) 2.1 이라는 상당히 낮은 처리량을 보여주고 있습니다. 또한 전체 요청 중 약 30%에 달하는 요청에서 에러가 발생했음을 알 수 있습니다.
![[스크린샷 2023-12-13 오후 5.02.49.png]]

로그를 확인해보니 커넥션 고갈로 인해 시간 내에 요청을 처리하지 못해 타임 아웃이 발생했음을 확인할 수 있었습니다.

다음은 개선 후의 리포트입니다.
![[개선후(1).png]]![[개선후(2).png]]

프로젝션을 이용해서 개선 한 후의 처리량은 평균 TPS 8.7로 개선전과 비교했을 때는 꽤나 상승하였습니다. 하지만 생각만큼 높은 처리량은 아니었습니다.

아직 부족한 점이 있다고 판단하여 `explain` 명령을 통해 개선 후 쿼리의 실행 계획을 살펴보았습니다.
![[explain(인덱스없음).png]]

쿼리의 성능과 관련된 내용은 `Extra` 컬럼을 통해서 확인할 수 있습니다. 여기서 주목해야 하는 메시지는 `Using filesort`입니다.

`Using filesort`는 `ORDER BY`가 인덱스를 사용하지 못하고 MySQL 서버가 별도의 메모리 버퍼에서 정렬을 수행하고 있다는 것을 의미합니다. 이는 상당한 부하가 걸리는 작업으로 쿼리 튜닝이나 적절한 인덱스를 생성해줘야 합니다.

## 인덱스 붙이기
---
보호 동물 목록 조회 시 정렬 조건은 JPA auditing에 의해서 자동 생성되는 `createdAt`을 이용하고 있습니다. 따라서 해당 컬럼에 인덱스를 생성해주었습니다.
```sql
alter table animal add index created_idx(created_at);
show index from animal;
```

![[스크린샷 2023-12-15 오후 5.20.03.png]]

데이터를 넣어줄 때 생성 시간을 다양하게 설정해주었기 때문에 높은 카디널리티를 가지는 것을 볼 수 있습니다. 실제 운영시에도 비슷한 수준일 것이라고 예상합니다.

동일한 쿼리에 대해 다시 `explain` 명령을 실행시키면 `extra` 컬럼의 메시지가 변경된 것을 확인할 수 있습니다.
![[explain(인덱스).png]]

`Backward index scan`은 인덱스 리프 노드의 오른쪽 페이지부터 왼쪽으로 스캔했음을 뜻하는 메시지입니다. 기존 `Using filesort`와 달리 인덱스를 이용해 빠른 속도로 스캔이 이루어졌음을 기대할 수 있습니다.
![[성능테스트(최종1).png]]
![[성능테스트(최종2).png]]

ngrinder를 이용해 성능 테스트를 수행해보면 평균 TPS 8.1에서 33.6으로 4배 이상 증가하였습니다. 인덱스만 적절하게 달아주었음에도 상당한 성능 향상을 확인할 수 있었습니다. 

이제 수만건, 수십만건에 대한 조회는 어느 정도의 개선이 이루어져 이전보다는 빠른 속도로 수행됩니다. 만일 데이터가 수백만 건이 넘어간다면 현재의 쿼리를 `no offset`으로 개선을 수행해야 할 것입니다.