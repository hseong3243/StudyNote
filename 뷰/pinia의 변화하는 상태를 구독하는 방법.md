스토어의 `$subscribe()` 메서드를 사용하여 상태의 변경 사항을 감시할 수 있다.
```javascript
mounted() {  
  this.linkBundleStore.$subscribe((mutation, state) => {  
    this.linkBundle = state.linkBundle;  
  })  
},
```

## 참고
---
https://pinia.vuejs.kr/core-concepts/state