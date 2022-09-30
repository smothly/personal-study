# SEC05 빅데이터 관리의 핵심 기술 MongoDB 구축

---

## 5-1 MongoDB Atlas 클라우드 구축

---

- MongoDB 사이트 들어가 클러스터 구축


## 5-2 MongoDB 접근 권한 설정 & Compass 셋업

---

- IP 허용
- User/pw 설정
- compass 다운로드하고 연결정보 입력

## 5-3 MongoDB CRUD

---

- database 생성
- collection 생성

```SQL
use ye
db.users.insertOne({name:"aaa", email: "aaa@naver.com"})
db.users.find()
db.users.updateOne({_id: ObjectId("aaaaaaaaaaaaa")}, {$set {name: "hello"}})
db.users.deleteOne({_id: ObjectId("aaaaaaaaaaaaa")})
```