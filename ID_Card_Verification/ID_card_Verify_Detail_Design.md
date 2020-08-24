### Data structure

1. **DataBase**

   ```mssql
   ALTER table ls_um_user_info_t
   ADD COLUMN verification_status varchar(30) DEFAULT 'UN_VERIFIED' COMMENT 'id verification status'
   ```

2. **DTO**

   ```java
   class UserInfoDTO {
     private VerficationStatus verificationStatus;
   }
   
   enum VerficationStatus {
     UN_VERIFIED, CHINA_ID_VERIFIED
   }
   ```



### API definition

1. **ID card verify** (new api)

   **POST** um/v1/info/idcard/verify

   *Verify ID card and upload result to redis as cache*

   ``` javascript
   role acccess: worker
   
   request:
   String idCardNumber
   String name
   
   response:
   data(boolean): true/false  
   HTTP CODE:200  //verified, result is in data
   HTTP CODE:400	 //request failed, need to retry
   ```

   

2. **ID card verify and update**

   **POST** um/v1/info/idcard/verify-and-update

   *Verify ID card and update ID card if verification passed*

   ```javascript
   role acccess: worker
   
   request:
   String idCardNumber
   String name
   
   response:
   data(boolean): true/false  
   HTTP CODE:200  //verified, result is in data
   HTTP CODE:400	 //request failed, need to retry
   ```



3. **Check IP region**

   **GET** um/v1/info/ip-region

   Find the real region by user's ip address

   ```javascript
   role acccess: all roles
   
   request:
   nothing
   
   response:
   data(String): "中国","美国",...
   HTTP CODE:200  //checked, result in field : data
   HTTP CODE:400	 //check fails, need to retry
   ```

   

3. **ID card update** (already exists)

   **POST** um/v1/info/idcard/update

   *Check verification status if idCardType is CHINA_RESIDENT_IDENTITY_CARD, return failed response if verification status is VERIFIED*

   

3. **Retrieve user info** (already exists)

   **GET** um/v1/info/get

   **GET** um/v1/info/get/id

   *return verificationStatus*



### Process Flow

1. **Complete User Info**
