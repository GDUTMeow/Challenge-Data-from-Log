# Data from Log

>[!important]
> 
> 数据集见 Release

**所有数据均为随机生成**

现在发生了一件很不幸的事情，数据库里面的数据丢失了，我们无法通过数据库来获取需要的内容，所幸的是，阿帕奇的 log 还在

## 题目要求

现在你需要做下面的这些事情

- 首先，你得从日志中还原出各个用户输入的个人信息，他们包含了
  - 用户名：只能由字母和数字组成
    - 正确示例：`admin`、`Luminoria`、`KeqingMoe`、`h3ll0World`
    - 错误示例：`ad_min`、`Zhang_san`
  - 姓名：全中文
    - 正确示例：`张三`、`李四`
    - 错误示例：`z三`、`李4`
  - 身份证号：固定为 18 位，符合国家关于身份证的校验规则（本题不设置性别校验）
  - 手机号码：为 11 位数字字符串，其中前三位固定在下面的集合中
    - `734, 735, 736, 737, 738, 739, 747, 748, 750, 751, 752, 757, 758, 759, 772, 778, 782, 783, 784, 787, 788, 795, 798, 730, 731, 732, 740, 745, 746, 755, 756, 766, 767, 771, 775, 776, 785, 786, 796, 733, 749, 753, 773, 774, 777, 780, 781, 789, 790, 791, 793, 799`
  - 密码：由数字和字母组成，没有符号
- 接着，你要对数据进行脱敏
  - 用户名：若只有两个字符则只对最后⼀位使⽤ \* 号代替，否则只保留第⼀位和最后⼀位字符，其余都⽤ \* 号代替，例如“ab”脱敏后就是“a\*”，“abcde”脱敏后就是“a\*\*\*e”。
  - 密码：对明文密码进行 md5 计算，例如 `123456` 计算后为 `e10adc3949ba59abbe56e057f20f883e`
  - 名字：⼆字姓名对最后⼀位使⽤ \* 号代替，三字即以上姓名除第⼀位和最后⼀位以外都使⽤ \* 号代替， 例如 `张三` 脱敏后就是 `张*`，`王不⼆` 脱敏后就是 `王*⼆`
  - 身份证号：只保留年份，其余都使⽤ \* 号代替，例如 `172865199108200356` 脱敏后就是 `******1991********`
  - 手机号码：对 4-7 位的地区编码使⽤ \* 号代替，例如 `74580417166` 脱敏后就是 `745****7166`
- 最后，你应该将脱敏后的所有数据按照 `username, password, name, idcard, phone` 对应 `用户名、密码、名字、身份证号、手机号码` 的顺序，将所有数据**按照 UTF8 编码方式**保存，文件的最后应该有一行空行
- 表头应该为 `username,password,name,idcard,phone`

## 注意

- 你的换行应该使用 `\n`，你可以在调试的时候使用 `print(repr(data))` 来确认这一点
- 你进行 md5 运算的文件内容中，你需要注意不要忘掉首行的表头

## 题解

本题数据来自 2024 羊城杯初赛

```python
import csv
import re
import urllib.parse
import hashlib
from tqdm import tqdm

with open("apache2/error.log") as f:
    data = f.readlines()


def write_to_csv(dataset: list, output_file: str):
    # 定义CSV表头
    headers = ["username", "password", "name", "idcard", "phone"]

    # 打开文件进行写入
    with open(output_file, "w", newline="", encoding="utf-8") as csvfile:
        writer = csv.writer(csvfile)

        # 写入表头
        writer.writerow(headers)

        # 写入数据
        for row in dataset:
            writer.writerow(row)


def firstFilter(data: list):
    pattern = re.compile(r"username=[^&\s]*.*")
    passwdList = []
    newdata = []
    temp_entry = None

    for line in tqdm(data):
        match = pattern.search(line)
        if match:
            if temp_entry and passwdList:
                temp_entry += f"&password={passwdList[-1]}"
                newdata.append(temp_entry)
            temp_entry = urllib.parse.unquote(match.group(0).strip())
            passwdList = []
        elif temp_entry:
            password = extract_and_clean_password(line)
            if password:
                passwdList.append(password)
                print(password)

    # 处理最后一个条目
    if temp_entry and passwdList:
        temp_entry += f"&password={passwdList[-1]}"
        newdata.append(temp_entry)

    return newdata


def extract_and_clean_password(log_line: str) -> str:
    # 匹配 \xe6 开头后的内容
    password_pattern = re.compile(r"\\xe6.*?:\s(.*)\\n")
    match = password_pattern.search(log_line)
    if match:
        # 提取出密码部分并移除无用的转义字符和空白字符
        password = match.group(1).replace("\\n", "").replace("\\", "").replace(" ", "")
        return password
    return None


def secondFilter(username, name, idcard, phone) -> bool:
    if (
        isUsernameValid(username)
        and isNameValid(name)
        and isNumberValid(idcard)
        and isPhoneValid(phone)
    ):
        return True
    return False


def extractInfo(line: str) -> tuple[str]:
    user_pattern = re.compile(r"username=(.*?)&")
    name_pattern = re.compile(r"name=(.*?)&")
    idcard_pattern = re.compile(r"idcard=(.*?)&")
    phone_pattern = re.compile(r"phone=(.*)&")
    password_pattern = re.compile(r"password=(.*)")

    username = user_pattern.search(line)[0].replace("username=", "").replace("&", "")
    name = (
        name_pattern.findall(line)[1].replace("name=", "").replace("&", "")
    )  # 因为username在name之前，所以第二个才是姓名
    idcard = idcard_pattern.search(line)[0].replace("idcard=", "").replace("&", "")
    phone = phone_pattern.search(line)[0].replace("phone=", "").replace("&", "")
    password = password_pattern.search(line)[0].replace("password=", "")
    return (username, name, idcard, phone, password)


def isNameValid(name: str) -> bool:
    pattern = re.compile(r"^[\u4e00-\u9fff]+$")
    return bool(pattern.match(name))


def isUsernameValid(username: str) -> bool:
    pattern = re.compile(r"^[a-zA-Z0-9]+$")
    return bool(pattern.match(username))


def isNumberValid(idNumber: str) -> bool:
    # 检查身份证号长度
    if len(idNumber) != 18:
        return False
    # 检查格式是否符合要求
    if not re.match(r"^\d{17}[\dX]$", idNumber):
        return False
    # 系数和校验码映射表
    coefficients = [7, 9, 10, 5, 8, 4, 2, 1, 6, 3, 7, 9, 10, 5, 8, 4, 2]
    check_digit_map = "10X98765432"
    # 计算校验码
    sum_result = sum(int(idNumber[i]) * coefficients[i] for i in range(17))
    remainder = sum_result % 11
    check_digit = check_digit_map[remainder]
    # 验证校验码是否匹配
    if idNumber[-1] != check_digit:
        return False
    return True


def isPhoneValid(phone: str) -> bool:
    if len(phone) != 11 or not phone.isdigit():
        return False

    valid_prefixes = {
        "734",
        "735",
        "736",
        "737",
        "738",
        "739",
        "747",
        "748",
        "750",
        "751",
        "752",
        "757",
        "758",
        "759",
        "772",
        "778",
        "782",
        "783",
        "784",
        "787",
        "788",
        "795",
        "798",
        "730",
        "731",
        "732",
        "740",
        "745",
        "746",
        "755",
        "756",
        "766",
        "767",
        "771",
        "775",
        "776",
        "785",
        "786",
        "796",
        "733",
        "749",
        "753",
        "773",
        "774",
        "777",
        "780",
        "781",
        "789",
        "790",
        "791",
        "793",
        "799",
    }

    prefix = phone[:3]
    if prefix not in valid_prefixes:
        return False

    return True


def generate_md5_hash(input_string: str) -> str:
    # 创建一个md5哈希对象
    md5_hash = hashlib.md5()

    # 更新哈希对象
    md5_hash.update(input_string.encode("utf-8"))

    # 获取32位MD5哈希值并转换为小写
    return md5_hash.hexdigest().lower()


def desensitize(username, name, idcard, phone, password):
    if len(username) == 2:
        deUsername = username[:1] + "*"
    else:
        deUsername = username[:1] + "*" * (len(username) - 2) + username[-1:]

    if len(name) == 2:
        deName = name[:1] + "*"
    else:
        deName = name[:1] + "*" * (len(name) - 2) + name[-1:]

    deIdcard = "*" * 6 + idcard[6:10] + "*" * 8
    dePhone = phone[:3] + "*" * 4 + phone[-4:]

    dePassword = generate_md5_hash(password)

    return (deUsername, deName, deIdcard, dePhone, dePassword)


validDataset = []
invalidDataset = []

if __name__ == "__main__":
    data = firstFilter(data)
    for line in tqdm(data):
        username, name, idcard, phone, password = extractInfo(line)
        if secondFilter(username, name, idcard, phone):  # 第二次筛选：筛选合法数据
            validDataset.append((username, name, idcard, phone, password))
        else:
            invalidDataset.append((username, name, idcard, phone, password))

    desensitizedDataset = []
    for line in tqdm(validDataset):
        username, name, idcard, phone, password = line
        username, name, idcard, phone, password = desensitize(
            username, name, idcard, phone, password
        )
        desensitizedDataset.append(
            (username, password, name, idcard, phone)
        )  # 调换位置，题目是username, password, name, idcard, phone

    write_to_csv(desensitizedDataset, "output.csv")
```

`flag{de95bf79eace2722b90fbecc35cab2ed}`