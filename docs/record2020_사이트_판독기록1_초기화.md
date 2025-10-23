2022년 이후 입국자들이 사용하는 record2.radiology.or.kr사이트에는 판독기록1에 전체삭제 버튼이 있어서 초기화가 쉽게 가능합니다.

2021년 이전 입국자들이 사용하는 record2020.radiology.or.kr사이트는 판독기록 전체 삭제 기능이 지원되지 않아 모두 삭제하고 새로올리고싶을때 어려움이 있습니다.

아래 방법을 사용하여 초기화하고 LogbookManager를 활용해 새로 올려보세요!

1. record2020 사이트에 접속하여 판독기록1로 들어갑니다.
2. 판독기록1 페이지 (기존 로그북이 올라간 페이지)에서 F12를 누릅니다.
3. 개발자도구창이 뜨면 콘솔(console)탭에 아래 스크립트를 복사해 붙여넣으려면 `Ctrl` + `V` (윈도우/크롬북) 또는 `⌘` + `V` (맥)을 누르고, 엔터(`Enter`) 키를 입력하세요.
```javascript
$("#check_all").prop("checked",true)
check_all()
function delete_member(){
    str = '';    i = 0;    $("input[name^=no]").each(function() {
        if($(this).attr("name").substring(0,3)=="no_") {
            if($(this).is(":checked")) {
                if (i==0) str = $(this).attr("name").substring(3);                else str += "," + $(this).attr("name").substring(3);                i++;            }
        }
    });    location.href='/record/record.kin?step=4&input_mode=delete&input_no='+str;}
delete_member()
```
4. `Enter`키를 입력하면 한페이지씩 삭제가됩니다. `Ctrl` + `V` 후 `Enter`를 반복하여 모든페이지를 삭제하세요.