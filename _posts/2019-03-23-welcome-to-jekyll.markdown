---
layout: post
title:  "这是个标题"
date:   2019-03-23 21:03:36 +0530
categories: Javascript NodeJS
---
这是正文

```javascript
const Razorpay = require('razorpay');

let rzp = Razorpay({
	key_id: 'KEY_ID',
	secret: 'name'
});

// capture request
rzp.capture(payment_id, cost)
	.then(function (data) {
		return 2;
	})
	
@RequestMapping(value = "/api/thirdparty/query_fund_info", produces = "application/json;charset=UTF-8")
public @ResponseBody
String queryUserFundInfo(@RequestParam(value = "uid") long uid
        , HttpServletRequest request) {
    FundInfo fundInfo = new FundInfo();
    fundInfo.setErrorCode(1);
    try {
        String s = serverListService.queryUserFundInfo(uid);
        String[] infoSet = s.split("\\|", -1);
        if(infoSet.length>=3)
        {
            fundInfo.setName(infoSet[0]);
            fundInfo.setLevel(Integer.parseInt(infoSet[1]));
            fundInfo.setFundInfo(infoSet[2]);
            fundInfo.setCareer(Integer.parseInt(infoSet[3]));
        }
        fundInfo.setErrorCode(0);
    } catch (Exception e) {
        fundInfo.setErrorCode(2);
        e.printStackTrace();
    }
    return JSON.toJSONString(fundInfo);
}
```

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
