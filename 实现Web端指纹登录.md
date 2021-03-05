https://mp.weixin.qq.com/s/TmYbXOAaZYIYupK9lNwUig

实现Web端指纹登录
前言
现在越来越多的笔记本电脑内置了指纹识别，用于快速从锁屏进入桌面，一些客户端的软件也支持通过指纹来认证用户身份。

前几天我在想，既然客户端软件能调用指纹设备，web端应该也可以调用，经过一番折腾后，终于实现了这个功能，并应用在了我的开源项目中。

本文就跟大家分享下我的实现思路以及过程，欢迎各位感兴趣的开发者阅读本文。

实现思路
浏览器提供了Web Authentication API, 我们可以利用这套API来调用用户的指纹设备来实现用户信息认证。

最终的实现效果视频，请移步：web端指纹登录

注册指纹
首先，我们需要拿到服务端返回的用户凭证，随后将用户凭证传给指纹设备，调起系统的指纹认证，认证通过后，回调函数会返回设备id与客户端信息，我们需要将这些信息保存在服务端，用于后面调用指纹设备来验证用户身份，从而实现登录。

接下来，我们总结下注册指纹的过程，如下所示：

用户使用其他方式在网站登录成功后，服务端返回用户凭证，将用户凭证保存到本地
检测客户端是否存在指纹设备
如果存在，将服务端返回的用户凭证与用户信息传递给指纹注册函数来创建指纹
身份认证成功，回调函数返回设备id与客户端信息，将设备id保存到本地
将设备id与客户端信息发送至服务端，将其存储到指定用户数据中。
⚠️注意：注册指纹只能工作在使用 https 连接，或是使用 localhost的网站中。

指纹认证
用户在我们网站授权指纹登录后，会将用户凭证与设备id保存在本地，当用户进入我们网站时，会从本地拿到这两条数据，提示它是否需要通过指纹来登录系统，同意之后则将设备id与用户凭证传给指纹设备，调起系统的指纹认证，认证通过后，调用登录接口，获取用户信息。

接下来，我们总结下指纹认证的过程，如下所示：

从本地获取用户凭证与设备id
检测客户端是否存在指纹设备
如果存在，将用户凭证与设备id传给指纹认证函数进行校验
身份认证成功，调用登录接口获取用户信息
⚠️注意：指纹认证只能工作在使用 https 连接，或是使用 localhost的网站中。

实现过程
上一个章节，我们捋清了指纹登录的具体实现思路，接下来我们来看下具体的实现过程与代码。

服务端实现
首先，我们需要在服务端写3个接口：获取TouchID、注册TouchID、指纹登录

获取TouchID
这个接口用于判断登录用户是否已经在本网站注册了指纹，如果已经注册则返回TouchID到客户端，方便用户下次登录。

controller层代码如下
    @ApiOperation(value = "获取TouchID", notes = "通过用户id获取指纹登录所需凭据")
    @CrossOrigin()
    @RequestMapping(value = "/getTouchID", method = RequestMethod.POST)
    public ResultVO<?> getTouchID(@ApiParam(name = "传入userId", required = true) @Valid @RequestBody GetTouchIdDto touchIdDto, @RequestHeader(value = "token") String token) {
        JSONObject result = userService.getTouchID(JwtUtil.getUserId(token));
        if (result.getEnum(ResultEnum.class, "code").getCode() == 0) {
            // touchId获取成功
            return ResultVOUtil.success(result.getString("touchId"));
        }
        // 返回错误信息
        return ResultVOUtil.error(result.getEnum(ResultEnum.class, "code").getCode(), result.getEnum(ResultEnum.class, "code").getMessage());
    }
复制代码
接口具体实现代码如下
    // 获取TouchID
    @Override
    public JSONObject getTouchID(String userId) {
        JSONObject returnResult = new JSONObject();
        // 根据当前用户id从数据库查询touchId
        User user = userMapper.getTouchId(userId);
        String touchId = user.getTouchId();
        if (touchId != null) {
           // touchId存在
            returnResult.put("code", ResultEnum.GET_TOUCHID_SUCCESS);
            returnResult.put("touchId", touchId);
            return returnResult;
        }
        // touchId不存在
        returnResult.put("code", ResultEnum.GET_TOUCHID_ERR);
        return returnResult;
    }
复制代码
注册TouchID
这个接口用于接收客户端指纹设备返回的TouchID与客户端信息，将获取到的信息保存到数据库的指定用户。

controller层代码如下
    @ApiOperation(value = "注册TouchID", notes = "保存客户端返回的touchid等信息")
    @CrossOrigin()
    @RequestMapping(value = "/registeredTouchID", method = RequestMethod.POST)
    public ResultVO<?> registeredTouchID(@ApiParam(name = "传入userId", required = true) @Valid @RequestBody SetTouchIdDto touchIdDto, @RequestHeader(value = "token") String token) {
        JSONObject result = userService.registeredTouchID(touchIdDto.getTouchId(), touchIdDto.getClientDataJson(), JwtUtil.getUserId(token));
        if (result.getEnum(ResultEnum.class, "code").getCode() == 0) {
            // touchId获取成功
            return ResultVOUtil.success(result.getString("data"));
        }
        // 返回错误信息
        return ResultVOUtil.error(result.getEnum(ResultEnum.class, "code").getCode(), result.getEnum(ResultEnum.class, "code").getMessage());
    }
复制代码
接口具体实现代码如下
    // 注册TouchID
    @Override
    public JSONObject registeredTouchID(String touchId, String clientDataJson, String userId) {
        JSONObject result = new JSONObject();
        User row = new User();
        row.setTouchId(touchId);
        row.setClientDataJson(clientDataJson);
        row.setUserId(userId);
       // 根据userId更新touchId与客户端信息
        int updateResult = userMapper.updateTouchId(row);
        if (updateResult>0) {
            result.put("code", ResultEnum.SET_TOUCHED_SUCCESS);
            result.put("data", "touch_id设置成功");
            return result;
        }
        result.put("code", ResultEnum.SET_TOUCHED_ERR);
        return result;
    }
复制代码
指纹登录
这个接口接收客户端发送的用户凭证与touchId，随后将其和数据库中的数据进行校验，返回用户信息。

controller层代码如下
    @ApiOperation(value = "指纹登录", notes = "通过touchId与用户凭证登录系统")
    @CrossOrigin()
    @RequestMapping(value = "/touchIdLogin", method = RequestMethod.POST)
    public ResultVO<?> touchIdLogin(@ApiParam(name = "传入Touch ID与用户凭证", required = true) @Valid @RequestBody TouchIDLoginDto touchIDLogin) {
        JSONObject result = userService.touchIdLogin(touchIDLogin.getTouchId(), touchIDLogin.getCertificate());
        return LoginUtil.getLoginResult(result);
    }
复制代码
接口具体实现代码如下
    // 指纹登录
    @Override
    public JSONObject touchIdLogin(String touchId, String certificate) {
        JSONObject returnResult = new JSONObject();
        User row = new User();
        row.setTouchId(touchId);
        row.setUuid(certificate);
        User user = userMapper.selectUserForTouchId(row);
        String userName = user.getUserName();
        String userId = user.getUserId();
        // 用户名为null则返回错误信息
        if (userName == null) {
            // 指纹认证失败
            returnResult.put("code", ResultEnum.TOUCHID_LOGIN_ERR);
            return returnResult;
        }
       // 指纹认证成功，返回用户信息至客户端
       // ... 此处代码省略，根据自己的需要返回用户信息即可 ...//
       returnResult.put("code", ResultEnum.LOGIN_SUCCESS);
       return returnResult;
    }
复制代码
前端实现
前端部分，需要将现有的登录逻辑和指纹认证相结合，我们需要实现两个函数：指纹注册、指纹登录。

指纹注册
这个函数我们需要接收3个参数：用户名、用户id、用户凭证，我们需要这三个参数来调用指纹设备来生成指纹，具体的实现代码如下：

    touchIDRegistered: async function(
      userName: string,
      userId: string,
      certificate: string
    ) {
      // 校验设备是否支持touchID
      const hasTouchID = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
      if (
        hasTouchID &&
        window.confirm("检测到您的设备支持指纹登录，是否启用？")
      ) {
        // 更新注册凭证
        this.touchIDOptions.publicKey.challenge = this.base64ToArrayBuffer(
          certificate
        );
        // 更新用户名、用户id
        this.touchIDOptions.publicKey.user.name = userName;
        this.touchIDOptions.publicKey.user.displayName = userName;
        this.touchIDOptions.publicKey.user.id = this.base64ToArrayBuffer(
          userId
        );
        // 调用指纹设备，创建指纹
        const publicKeyCredential = await navigator.credentials.create(
          this.touchIDOptions
        );
        if (publicKeyCredential && "rawId" in publicKeyCredential) {
          // 将rowId转为base64
          const rawId = publicKeyCredential["rawId"];
          const touchId = this.arrayBufferToBase64(rawId);
          const response = publicKeyCredential["response"];
          // 获取客户端信息
          const clientDataJSON = this.arrayBufferToString(
            response["clientDataJSON"]
          );
          // 调用注册TouchID接口
          this.$api.touchIdLogingAPI
            .registeredTouchID({
              touchId: touchId,
              clientDataJson: clientDataJSON
            })
            .then((res: responseDataType<string>) => {
              if (res.code === 0) {
                // 保存touchId用于指纹登录
                localStorage.setItem("touchId", touchId);
                return;
              }
              alert(res.msg);
            });
        }
      }
    }
复制代码
上面函数中在创建指纹时，用到了一个对象，它是创建指纹必须要传的，它的定义以及每个参数的解释如下所示：

const touchIDOptions = {
  publicKey: {
    rp: { name: "chat-system" }, // 网站信息
    user: {
      name: "", // 用户名
      id: "", // 用户id(ArrayBuffer)
      displayName: "" // 用户名
    },
    pubKeyCredParams: [
      {
        type: "public-key",
        alg: -7 // 接受的算法
      }
    ],
    challenge: "", // 凭证(touchIDOptions)
    authenticatorSelection: {
      authenticatorAttachment: "platform"
    }
  }
}
复制代码
由于touchIDOptions中，有的参数需要ArrayBuffer类型，我们数据库保存的数据是base64格式的，因此我们需要实现base64与ArrayBuffer之间相互转换的函数，实现代码如下：

    base64ToArrayBuffer: function(base64: string) {
      const binaryString = window.atob(base64);
      const len = binaryString.length;
      const bytes = new Uint8Array(len);
      for (let i = 0; i < len; i++) {
        bytes[i] = binaryString.charCodeAt(i);
      }
      return bytes.buffer;
    },
    arrayBufferToBase64: function(buffer: ArrayBuffer) {
      let binary = "";
      const bytes = new Uint8Array(buffer);
      const len = bytes.byteLength;
      for (let i = 0; i < len; i++) {
        binary += String.fromCharCode(bytes[i]);
      }
      return window.btoa(binary);
    }
复制代码
指纹认证通过后，会在回调函数中返回客户端信息，数据类型是ArrayBuffer，数据库需要的格式是string类型，因此我们需要实现ArrayBuffer转string的函数，实现代码如下：

    arrayBufferToString: function(buffer: ArrayBuffer) {
      let binary = "";
      const bytes = new Uint8Array(buffer);
      const len = bytes.byteLength;
      for (let i = 0; i < len; i++) {
        binary += String.fromCharCode(bytes[i]);
      }
      return binary;
    }
复制代码
注意⚠️：用户凭证中不能包含 _ 和 **-**这两个字符，否则base64ToArrayBuffer函数将无法成功转换。

指纹登录
这个函数接收2个参数：用户凭证、设备id，我们会通过这两个参数来调起客户端的指纹设备来验证身份，具体的实现代码如下：

touchIDLogin: async function(certificate: string, touchId: string) {
      // 校验设备是否支持touchID
      const hasTouchID = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
      if (hasTouchID) {
        // 更新登录凭证
        this.touchIDLoginOptions.publicKey.challenge = this.base64ToArrayBuffer(
          certificate
        );
        // 更新touchID
        this.touchIDLoginOptions.publicKey.allowCredentials[0].id = this.base64ToArrayBuffer(
          touchId
        );
        // 开始校验指纹
        await navigator.credentials.get(this.touchIDLoginOptions);
        // 调用指纹登录接口
        this.$api.touchIdLogingAPI
          .touchIdLogin({
            touchId: touchId,
            certificate: certificate
          })
          .then((res: responseDataType) => {
            if (res.code == 0) {
              // 存储当前用户信息
              localStorage.setItem("token", res.data.token);
              localStorage.setItem("refreshToken", res.data.refreshToken);
              localStorage.setItem("profilePicture", res.data.avatarSrc);
              localStorage.setItem("userID", res.data.userID);
              localStorage.setItem("username", res.data.username);
              const certificate = res.data.certificate;
              localStorage.setItem("certificate", certificate);
              // 跳转消息组件
              this.$router.push({
                name: "message"
              });
              return;
            }
            // 切回登录界面
            this.isLoginStatus = loginStatusEnum.NOT_LOGGED_IN;
            alert(res.msg);
          });
      }
    }
复制代码
注意⚠️：注册新的指纹后，旧的Touch id就会失效，只能通过新的Touch ID来登录，否则系统无法调起指纹设备，会报错：认证出了点问题。

整合现有登录逻辑
完成上述步骤后，我们已经实现了整个指纹的注册、登录的逻辑，接下来我们来看看如何将其与现有登录进行整合。

调用指纹注册
当用户使用用户名、密码或者第三方平台授权登录成功后，我们就调用指纹注册函数，提示用户是否对本网站进行授权，实现代码如下：

 authLogin: function(state: string, code: string, platform: string) {
      this.$api.authLoginAPI
        .authorizeLogin({
          state: state,
          code: code,
          platform: platform
        })
        .then(async (res: responseDataType) => {
          if (res.code == 0) {
            //  ... 授权登录成功，其他代码省略 ... //
            // 保存用户凭证，用于指纹登录
            const certificate = res.data.certificate;
            localStorage.setItem("certificate", certificate);
            // 校验设备是否支持touchID
            const hasTouchID = await PublicKeyCredential.isUserVerifyingPlatformAuthenticatorAvailable();
            if (hasTouchID) {
              //  ... 其他代码省略 ... //
              // 获取Touch ID，检测用户是否已授权本网站指纹登录
              this.$api.touchIdLogingAPI
                .getTouchID({
                  userId: userId
                })
                .then(async (res: responseDataType) => {
                  if (res.code !== 0) {
                    // touchId不存在, 询问用户是否注册touchId
                    await this.touchIDRegistered(username, userId, certificate);
                  }
                  // 保存touchid
                  localStorage.setItem("touchId", res.data);
                  // 跳转消息组件
                  await this.$router.push({
                    name: "message"
                  });
                });
              return;
            }
            // 设备不支持touchID，直接跳转消息组件
            await this.$router.push({
              name: "message"
            });
            return;
          }
          // 登录失败，切回登录界面
          this.isLoginStatus = loginStatusEnum.NOT_LOGGED_IN;
          alert(res.msg);
        });
    }
复制代码
最终的效果如下所示：





每次第三方平台授权登录时都会检测当前用户是否已授权本网站，如果已授权则将Touch ID保存至本地，用于通过指纹直接登录。

调用指纹登录
当登录页面加载完毕1s后，我们从用户本地取出用户凭证与Touch ID，如果存在则提示用户是否需要通过指纹来登录系统，具体代码如下所示：

  mounted() {
    const touchId = localStorage.getItem("touchId");
    const certificate = localStorage.getItem("certificate");
    // 如果touchId存在，则调用指纹登录
    if (touchId && certificate) {
      // 提示用户是否需要touchId登录
      setTimeout(() => {
        if (window.confirm("您已授权本网站通过指纹登录，是否立即登录？")) {
          this.touchIDLogin(certificate, touchId);
        }
      }, 1000);
    }
  }
复制代码
最终效果如下所示：





项目地址
本文代码的完整地址请移步：Login.vue

在线体验地址：chat-system
项目GitHub地址：chat-system-github
写在最后
最近打算换工作，有没有广州这边的公司可以内推我呀😁，我的微信：Baymax-kt

文中如有错误，欢迎在评论区指正，如果这篇文章帮到了你，欢迎点赞和关注😊
本文首发于掘金，未经许可禁止转载💌
