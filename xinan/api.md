# <center>信息安全竞赛后台API</center>

## 后台服务器相关信息

    IP 不能说的秘密
    PORT 8000
    统一采用POST请求

## 后台用户相关接口

#### 用户创建

    /signup
    接受参数
        必须包含参数
            user_id (str) 注册用户名
            user_pwd (str) 注册用户密码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 用户登录

    /login
    接受参数
        必须包含参数
            user_id (str) 登录用户名
            user_pwd (str) 登录用户密码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        token (str)
            若用户登录成功则会包含该字段，为用户的 cookies_id

#### 用户退出登录

    /logout
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 修改用户密码

    /changeuserpwd
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            old_pwd (str) 用户现在所使用的密码
            new_pwd (str) 用户的新密码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 修改用户设置信息

    /changeuserinfo
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
        可选参数
            base_privacy (list of str) 用户的基础隐私设置
                有效值包括
                    address
                    phone
                    bankcard
                    car_number
                    org
                    email
                    idcard
                    time
                    url
                    name
                    stucard
                    contract
                    business_licence
                    serial_number
                    face
            scene_privacy (list of str) 用户的地点隐私设置
                有效值包括
                    bedroom
                    chemistry_lab
                    conference_center
                    conference_room
                    office_cubicles
                    airport_terminal
                    bank_vault
                    banquet_hall
                    bathroom
                    biology_laboratory
                    childs_room
                    dining_hall
                    dining_room
                    dorm_room
                    dressing_room
                    embassy
                    gymnasium
                    home_office
                    hospital_room
                    legislative_chamber
                    physics_laboratory
                    restaurant
                    server_room
            phone_num (str) 用户的手机号码
            idcard_num (str) 用户的身份证号码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 获得用户信息

    /getuserinfo
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        user (dict)
            若成功才会包含该字段
            userID (str) 用户名
            base_privacy (list of str) 用户的基础隐私设置
            scene_privacy (list of str) 用户的地点隐私设置
            phone_num (str) 用户的手机号码
            idcard_num (str) 用户的身份证号码

#### 用户添加自己的人脸

    /addface
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            suffix (str) 人脸图片的后缀
            content (str) 人脸图片的数据
                需要首先经过gzip压缩，之后再通过base64编码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 用户删除自己的人脸

    /deleteface
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            image_id (str) 用户要删除的人脸的图片ID
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 用户获取已经添加的人脸信息

    /getfaces
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        faces (list of dict) 用户的人脸照信息列表
            每个人脸信息包含以下字段
                imageID (str) 用户获取图片二进制数据的图片ID
                date (int) 图片添加时间毫秒数

#### 用户添加相册照片

    /addphoto
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            suffix (str) 照片的后缀
            text (str) 照片的文字描述
            content (str) 照片的数据
                需要首先经过gzip压缩，之后再通过base64编码
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        imageID (str) 该照片的图片ID
        face_to_loc (list of list) 需要打码的人脸位置 [xmin, ymin, xmax, ymax];原来是(ymin, xmax, ymax, xmin)
        privacy_loc (dict of list) 
            key (str) 选取的标识
            value (list) 需要选取的隐私信息框(顺时针)
        scene (str) 照片的场景名
        score (float) 隐私评级分数

#### 用户选定照片打码位置ID

    /ensureprivacyloc
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            image_id (str) 该照片的图片ID
            index (list of str) 隐私框标识ID
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 用户删除相册照片

    /deletephoto
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            image_id (str) 用户要删除的照片的图片ID
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 用户获取自己照片信息列表

    /getphotos
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            begin (int) 按添加时间排序的开始坐标 
            limit (int) 返回的照片信息的最大个数
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        photos (list of dict)
            imageID (str) 图片ID
            privacy_index (list of str) 需要隐私框的标识列表
            privacy_loc (dict of list)
                key (str) 选取的标识
                value (list) 需要选取的隐私信息框(顺时针)
            text (str) 照片文字描述
            face_to_loc (list of list) 需要打码的人脸位置 [xmin, ymin, xmax, ymax];原来是(ymin, xmax, ymax, xmin)
            scene (str) 照片的场景名
            score (float) 隐私评级分数
            date (int) 日期毫秒数

#### 用户唯一可以获取图片二进制数据接口

    /getimage
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            image_id (str) 用户要获取的图片的ID
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        suffix (str) 图片的后缀
        content (str) 图片的数据
            首先经过gzip压缩，后经过base64编码
            要获取原始数据需要先经过base64解码，之后通过gzip解压缩
        date (int) 日期毫秒数
