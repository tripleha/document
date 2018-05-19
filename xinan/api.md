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
            base_privacy (list) 用户的基础隐私设置
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
            scene_privacy (list) 用户的地点隐私设置
                    bedroom
                    chemistry_lab
                    conference_center
                    conference_room
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败

#### 获得用户设置信息

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
            base_privacy (list) 用户的基础隐私设置
            scene_privacy (list) 用户的地点隐私设置

#### 用户添加自己的人脸

    /addface
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
            suffix (str) 人脸图片的后缀
            content (str) 人脸图片的二进制数据
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

#### 用户获取已经添加的人脸ID

    /getfaces
    接受参数
        必须包含参数
            token (str) 用户登录时所获得的 cookies_id
    返回结果
        code (int)
            1 表示 成功
            0 表示 失败
        face_ids (list) 用户的人脸ID列表

#### 用户获取图片二进制数据

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
        content (str) 图片的二进制数据
        date (int) 日期毫秒数
