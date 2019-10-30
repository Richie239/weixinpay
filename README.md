# weixinpay
微信支付源码

1. 后端获取秘钥：
   ```
    @Override
    public Object pay(HttpServletRequest request,String cardno, double total,String mchid,String openid,String canteenid){

        MerchantEntity merchantEntity= merchantRedis.get(mchid);
        if (AssertUtil.isEmpty(merchantEntity)){
            merchantEntity=  merchantService.getMerchantbyMchId(mchid);
            merchantRedis.saveOrUpdate(merchantEntity);
        }
        //①、生成商户商户订单号
        String x = DateUtil.format(new Date(), new SimpleDateFormat("yyyyMMddHHmmss"));
        String voucherId1 = "fkpay_" + x; //生成商户订单号
        Map<String, Object> map = new HashMap<>();
        Unifiedorder unifiedorder = new Unifiedorder();
        unifiedorder.setAppid(merchantEntity.getAppid());
        unifiedorder.setMch_id(mchid);
        unifiedorder.setNonce_str(UUID.randomUUID().toString().toString().replace("-", ""));
        unifiedorder.setOpenid(openid);
        unifiedorder.setBody("饭卡充值");
        unifiedorder.setOut_trade_no(voucherId1);
        //化为分去掉小数点后面的数字
        DecimalFormat decimalFormat = new DecimalFormat("###################.###########");
        unifiedorder.setTotal_fee(String.valueOf(decimalFormat.format(total*100)));//单位分
        unifiedorder.setSpbill_create_ip(request.getRemoteAddr());//IP
        unifiedorder.setNotify_url(merchantEntity.getNotifyUrl());
        unifiedorder.setTrade_type("JSAPI");//JSAPI，NATIVE，APP，WAP
        //统一下单，生成预支付订单
        UnifiedorderResult unifiedorderResult = PayMchAPI.payUnifiedorder(unifiedorder,merchantEntity.getKey());

        //@since 2.8.5  API返回数据签名验证
        if(unifiedorderResult.getSign_status() !=null && unifiedorderResult.getSign_status()){
            String json = PayUtil.generateMchPayJsRequestJson(unifiedorderResult.getPrepay_id(), merchantEntity.getAppid(), merchantEntity.getKey());
            map.put("param", json);
            map.put("outTradeNo", voucherId1);

            //本地生成一条订单
            WxOrderEntity wxOrderEntity =new WxOrderEntity();
                wxOrderEntity.setOrderId(UUID.randomUUID().toString());
                wxOrderEntity.setCardNo(cardno);
                wxOrderEntity.setMchId(mchid);
                wxOrderEntity.setBody("饭卡充值");
                wxOrderEntity.setType("微信小程序");
                wxOrderEntity.setIp(request.getRemoteAddr());
                wxOrderEntity.setOutTradeNo(voucherId1);
                wxOrderEntity.setTotal(String.valueOf(total));
                wxOrderEntity.setStatuses("0");  //微信订单充值中
                wxOrderEntity.setOpenId(openid);
                wxOrderEntity.setCanteenId(canteenid);
            wxOrderService.insert(wxOrderEntity);

            //返回签名验证
            logger.info(json);
            return map;
        }else{
            throw new RestException("支付失败。请联系管理员。");
        }
    }
   
    ```
  2. 前端vue代码
     ```
     onBridgeReady(obj, tradeNo) {
     var _this = this
     wx.requestPayment({
      timeStamp: obj.timeStamp,
      nonceStr: obj.nonceStr,
      package: obj.package,
      signType: obj.signType,
      paySign: obj.paySign,
      success: function(res) {
        let url = `carpay/payOrderquery?outtradeno=${tradeNo}&mchid=${_this.data.mchid}`
        https.request(url, '', 'POST', res => {
          console.log(res)
          wx.showModal({
            title: '提示',
            content: '支付成功',
            showCancel: false,
            confirmColor: '#DF222E',
            success: () => {
              //更新饭卡支付完成后的数据
              app.getcardinfo('', '', _this.data.openId, res => {
                if (res && res.cardNo) {
                  _this.setData({
                    money: res.money,
                    cardNo: res.cardNo,
                    workNo: `${res.jobnum} (${res.username})`,
                    canteen: res.canteenname,
                    canteenid: res.canteenId,
                    mchid: res.mchid
                  })
                }
              })
            }
          })
        })
      },
      fail: function(res) {
        console.log('支付失败', res)
        if (res.errMsg) {
          wx.showToast({
            title: '支付失败',
            icon: 'none'
          })
        }
      },
      complete: function(res) {
        // console.log(4455, res)
      }
    })
  },
   ```  
