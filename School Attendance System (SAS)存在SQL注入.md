## 基本信息

- 仓库：`https://github.com/0mehedihasan/sas`
- Commit：`730de5a457eb4e4fe37a25b83e916bd9f1806980`

`Admin/createClass.php` 的删除操作在失效守卫后继续执行，并且把 `Id` 拼入 SQL：

```php
if (isset($_GET['Id']) && isset($_GET['action']) && $_GET['action'] == "delete" && isset($_GET['dbKey'])) {
  $Id = $_GET['Id'];
  $dbKey = $_GET['dbKey'];
  ...
  if ($dbConnection) {
    $query = mysqli_query($dbConnection, "DELETE FROM tblclass WHERE Id='$Id'");
  }
}
```

`Admin/createSessionTerm.php` 多处直接拼接请求字段，未登录也会执行：

```php
$sessionName=$_POST['sessionName'];
$termId=$_POST['termId'];
$query=mysqli_query($conn,"select * from tblsessionterm where sessionName ='$sessionName' and termId = '$termId'");
$query=mysqli_query($conn,"insert into tblsessionterm(sessionName,termId,isActive,dateCreated) value('$sessionName','$termId','0','$dateCreated')");
```

```php
$Id= $_GET['Id'];
$query=mysqli_query($conn,"select * from tblsessionterm where Id ='$Id'");
$query=mysqli_query($conn,"update tblsessionterm set sessionName='$sessionName',termId='$termId',isActive='0' where Id='$Id'");
$query = mysqli_query($conn,"DELETE FROM tblsessionterm WHERE Id='$Id'");
$que=mysqli_query($conn,"update tblsessionterm set isActive='1' where Id='$Id'");
```

### 攻击场景

1. 攻击者可以直接向 `/Admin/createStudents.php` 创建学生账号，默认密码为 `12345`，污染学生身份数据。
2. 向 `/Admin/createClass.php?action=delete&dbKey=...&Id=...` 或 `/Admin/createClassArms.php?action=delete&dbKey=...&Id=...` 发送请求，删除班级或班臂数据。通过拼接参数影响 SQL 条件从而造成sql注入


### 防御建议

1. 使用 PDO 预编译语句
2. 使用 MySQLi 预编译语句
3. 不要拼接字符串