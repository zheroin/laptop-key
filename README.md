# springboot-keycloak-openldap

## Goal

The goal of this project is to create a simple REST API and securing it with Keycloak. Furthermore, Keycloak's users will be loaded from a OpenLDAP server.

## Start Environment

### Docker Compose

1. Open one terminal

2. Inside `/springboot-keycloak-openldap/dev` folder run
```
docker-compose up
```

### LDAP

1. Access the link
```
https://localhost:6443
```

2. Login with the credentials
```
Login DN: cn=admin,dc=mycompany,dc=com
Password: admin
```

3. Import the file `ldap-mycompany-com.ldif`

This file has already a pre-defined structure for mycompany.com.
Basically, it has 2 groups (employees and clients) and 3 users (Bill Gates, Steve Jobs and Mark Cuban). Besides, it is defined that Bill Gates and Steve Jobs belong to employees group and Mark Cuban belongs to clients group.
```
Bill Gates > username: bgates, password: 123
Steve Jobs > username: sjobs, password: 123
Mark Cuban > username: mcuban, password: 123
```

### Keycloak

1. Access the link
```
http://localhost:8181
```

2. Login with the credentials
```
Username: admin
Password: admin
```

3. Create a new Realm
- Go to top-left corner where `Master` realm is. A blue button `Add realm` will appear. Click on it.
- On `Name` field, write `company-services`. Click on `Create`.

4. Create a new Client
- Click on `Clients` menu on the left.
- Click `Create` button.
- On `Client ID` field type `springboot-keycloak-openldap`.
- Click on `Save`.
- On `Settings` tab, set the `Access Type` to `confidential`
- Still on `Settings` tab, set the `Valid Redirect URIs` to `http://localhost:8080/*`
- Click on `Save`.
- Go to `Credentials` tab. Copy the value on `Secret` field and set this value on `springboot-keycloak-openldap`application.yaml (property `keycloak.credentials.secret`).

5. Create a new Role
- Click on `Roles` menu on the left.
- Click `Add Role` button.
- On `Role Name` type `user`.
- Click on `Save`.

6. LDAP Integration
- Click on the `User Federation` menu on the left.
- Select `ldap`.
- On `Vendor` field select `Other`
- On `Connection URL` type `ldap://<ldap-service_ip-address>:389`. To get the ldap-service ip address run the following docker command:
```
docker inspect -f "{{ .NetworkSettings.Networks.dev_default.IPAddress }}" ldap-service
```
- Click on `Test connection` to check if it is ok.
- On `Users DN` type `ou=users,dc=mycompany,dc=com`
- On `Bind DN` type `cn=admin,dc=mycompany,dc=com`
- On `Bind Credential` set `admin`
- Click on `Test authentication` to check if it is ok.
- Click on `Save`.
- Click on `Synchronize all users`.

7. Configure users imported
- Click on `Users` menu on the left.
- Click on `View all users`. 3 users will be shown.
- Edit user `bgates`.
- Go to `Role Mappings` tab and add role `user` to him.
- Do the same for the user `sjobs`.
- Let's leave `mcuban` without `user` role.

### Spring Boot Application

1. Open a new terminal

2. Start `springboot-keycloak-openldap` application

In `springboot-keycloak-openldap` root folder, run those 2 commands:
```
mvn clean package -DskipTests
java -jar target/springboot-keycloak-openldap-0.0.1-SNAPSHOT.jar
```

## Test

1. Open a new terminal

2. Call the endpoint `/api/public` using the cURL command bellow.
```
curl 'http://localhost:8080/api/public'
```
It will return:
```
It is public.
```

3. Try to call the endpoint `/api/private` using the cURL command bellow.
``` 
curl 'http://localhost:8080/api/private'
```
It will return nothing.

4. Export to a variable the client secret generated by Keycloak for `springboot-keycloak-openldap`.
```
export KEYCLOAK_CLIENT_SECRET=<keycloak-client-secret>
```

5. Run the command bellow to get an access token for `bgates` user.
```
MYCOMPANY_BGATES_ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" -d "username=bgates" -d 'password=123' \
  -d 'grant_type=password' -d 'client_secret='$KEYCLOAK_CLIENT_SECRET \
  -d 'client_id=springboot-keycloak-openldap' | jq -r .access_token)
```

6. Call the endpoint `/api/private` using the cURL command bellow.
```
curl -X GET -H 'authorization: Bearer '$MYCOMPANY_BGATES_ACCESS_TOKEN 'http://localhost:8080/api/private'
```
It will return:
```
bgates, it is private.
```

7. Run the command bellow to get an access token for `mcuban` user.
```
MYCOMPANY_MCUBAN_ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" -d "username=mcuban" -d 'password=123' \
  -d 'grant_type=password' -d 'client_secret='$KEYCLOAK_CLIENT_SECRET \
  -d 'client_id=springboot-keycloak-openldap' | jq -r .access_token )
```

8. Try to call the endpoint `/api/private` using the cURL command bellow.
```
curl -X GET -H 'authorization: Bearer '$MYCOMPANY_MCUBAN_ACCESS_TOKEN 'http://localhost:8080/api/private'
```
As mcuban doesn't have the `user` role, he cannot access this endpoint. The endpoint return is:
```
"status":403,"error":"Forbidden","message":"Access is denied"
```

9. Go to Keycloak and add the role `user` to the `mcuban` user.

10. Run the command bellow to get a new access token for `mcuban` user.
```
MYCOMPANY_MCUBAN_ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" -d "username=mcuban" -d 'password=123' \
  -d 'grant_type=password' -d 'client_secret='$KEYCLOAK_CLIENT_SECRET \
  -d 'client_id=springboot-keycloak-openldap' | jq -r .access_token )
```

11. Call again the endpoint `/api/private` using the cURL command bellow.
```
curl -X GET -H 'authorization: Bearer '$MYCOMPANY_MCUBAN_ACCESS_TOKEN 'http://localhost:8080/api/private'
```
It will return:
```
mcuban, it is private.
```