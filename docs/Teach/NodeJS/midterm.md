---
slug: nodejs-midterm
title: NodeJS midterm
authors: [me]
---

<!-- truncate -->

# Kiểm tra giữa kỳ

![meme](image.png)

Khởi tạo một project mới và thực hiện những yêu cầu sau:

## Cấu hình (3 đ)

1. **Cài đặt thư viện & cấu hình nodemon:**

   1. Môi trường dev (<u>devDependencies</u>): `nodemon`
   2. Môi trường production (<u>dependencies</u>): `dotenv`, `express`, `jsonwebtoken`
   3. Cấu hình `nodemon` trong scripts section của `package.json` (Kết quả mong muốn: gõ `npm run dev` để chạy `nodemon`)

2. **Tạo file `.env` với nội dung sau:**

```yaml
API_Token = "ORR7z3KMKceWMK5xIFUNLlrFceXN"
JWT_Secret = "toYfwSSniETCVTs6"
```

3. **Tạo router tương ứng với từng mục:**
   - `simple`
   - `posts`
   - `auth`

## Triển khai API (7 đ)

### `simple` router

1. **`/submit-formdata`**
   - Method: POST
   - Header:
     - Content-type: application/x-www-form-urlencoded
   - Body: `name=John&age=18`
   - Response:
     - 200:
       ```json
       {
         "data": {
           "name": "John",
           "age": 18
         }
       }
       ```
     - 400: Nếu `body` không đủ thông tin `name` và `age`
2. **`/submit-json`**
   - Method: POST
   - Header:
     - Content-type: application/json
   - Body: `{"name": "Bob", "class": "NodeJS"}`
   - Response:
     - 200:
       ```json
       {
         "data": {
           "name": "Bob",
           "class": "NodeJS"
         }
       }
       ```
     - 400: Nếu `body` không đủ thông tin `name` và `age`
3. **`/get-data`**
   - Method: GET
   - Header:
     - X-API-KEY: [API_Token]
   - Response:
     - 200:
       ```json
       {
         "message": "Authorized"
       }
       ```
     - 401: Nếu `request` không có API_Token / API_Token không đúng

### `auth` router

:::info JWT Structure

- **Expire:** 5mins
- **Payload:**

```json
{
  "email": "<user-email>",
  "role": "<user-role>"
}
```

:::

1. **`/register`:**

   - Method: POST
   - Header:
     - Content-type: application/json
   - Body: `{"username": "<your-user-name>", "password": "<your-password>", "role": "<user-role>", "email": "<user-email>"}`
   - Response:
     - 200:
       ```json
       {
         "message": "Account created"
       }
       ```
     - 400: Nếu `body` không đủ thông tin

2. **`/login`:**
   - Method: POST
   - Header:
     - Content-type: application/json
   - Body: `{"username": "<exist-username>", "password": "<user-password>}`
   - Response:
     - 200:
       ```json
       {
         "token": "<JWT Token>"
       }
       ```
     - 400: Nếu `body` không đủ thông tin

:::tip JWT sign

```javascript
const token = jwt.sign(payload, SECRET_KEY, { expiresIn: "1h" });
```

:::

### `posts` router

:::info Post class

```json
{
	"id": int #increment,
	"title": string,
	"description": string,
	"author_email": string #user-email
}
```

:::

1. **`/`**
   - Method: GET
   - Header:
     - Authorization: `Bearer <jwt-token>`
   - Response:
     - 200:
       ```json
       		{
       			"data": [list of post]
       		}
       ```
     - 401: Nếu `request` không có token / token bị hết hạn
2. **`/`**
   - Method: POST
   - Header:
     - Authorization: `Bearer <jwt-token>`
   - Body: `{"title": "<title>", "description" : "<description>"}`
   - Response:
     - 200:
       ```json
       {
         "message": "created"
       }
       ```
     - 401: Nếu `request` không có token / token bị hết hạn
     - 400: Nếu `body` bị thiếu thông tin

:::tip JWT verify

```javascript
jwt.verify(token, SECRET_KEY, (err, user) => {
  /*callback*/
});
```

:::

## Nộp bài

- [Link nộp bài](https://nhgeduvn-my.sharepoint.com/:f:/g/personal/locnpv_giadinh_edu_vn/IgBC_CtrIXIZSLeqV0UgyTM9AS7ArJUW-rBOlRk0MMbB56I?e=cCAZ1t) (yêu cầu đăng nhập tài khoản của trường)

- Nén file và đặt tên theo cú pháp: <span style={{backgroundColor: 'red'}}>MSSV_Ten</span>

:::danger Chỉ nộp những file sau đây

1. File chứa code (server.js, routers)
2. package.json
3. .env

**Notes:** Nộp sai file / định dạng sẽ bị hủy kết quả thi
:::
