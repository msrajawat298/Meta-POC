- [Access Token Debugger](https://developers.facebook.com/tools/debug/accesstoken/)
- [Decode Time](https://dencode.com/en/date)
- [Meta Graph API Explorer](https://developers.facebook.com/tools/explorer/)
- [Long-Lived Access Tokens](https://developers.facebook.com/documentation/facebook-login/guides/access-tokens/get-long-lived#generate-long-lived-token)

- Issue Date of Token : Wed Jun 17 16:08:12 2026
- Expires : Sun Aug 16 16:08:12 2026
- "data_access_expires_at": 1787745334,
- "issued_at": 1781891402,

- Use this endpoint to generate the access_token Every 45 days for safer side: https://graph.facebook.com/me/accounts?access_token={{USER_ACCESS_TOKEN}}
- Use this endpoint to fetch the details : https://graph.facebook.com/v18.0/17841411615424388/media?fields=id%2Ccaption%2Cmedia_type%2Cmedia_url%2Cthumbnail_url%2Cpermalink%2Ctimestamp&access_token={{PAGE_ACCESS_TOKEN}}
