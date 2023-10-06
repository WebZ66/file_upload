# 文件上传

## 核心逻辑

客户端                                      -------------->(网络协议http) -------------->                                                服务器

样式、交互逻辑、运算                       消息格式、传输方式                                                   存储、安全、访问控制



**上传文件的`第一步`，也是最重要的一步，`调试接口`**。因为文件上传一定是发起一个http请求，同时把文件捎带过去。

(常用的工具postman、apifox)



## 上传逻辑界面

分为三部分

- 选取文件界面

  ![image-20231002141610594](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231002141610594.png) 

  ```vue
      <div v-if="flag == 'select'" class="upload-select" @click="handleSelect">
        <YkIcon name="jiahao" color="black"></YkIcon>
        <input
          ref="selectFile"
          id="select"
          type="file"
          @change="selectFileChange"
        />
      </div>
  ```

  > 唯一一个重点就是，input file需要让其display:none,点击容器时，自己手动触发input的click事件

  ```js
  function handleSelect() {
    const select = document.querySelector('#select') as HTMLInputElement
    select.click()
  }
  ```

- 上传进度界面

  ![image-20231002141821308](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231002141821308.png) 

  上传进度是最复杂的界面。

  - 首先需要实现一个进度条，可以使用css变量的方式来实现。代码如下：

    ```js
    //通过css变量，用js动态控制伪元素的宽度
    const progressNumber = ref(0)
    const progLiner = ref<HTMLDivElement | null>(null)
    function setProgress(value: number) {
      let v = 0.6 * value + 'px'
      progLiner.value?.style.setProperty('--progress', v)
    }
    ```

  ```vue
      <div v-else-if="flag == 'progress'" class="upload-progress">
        <div class="progress-number">{{ progressNumber }}%</div>
        <div ref="progLiner" class="progress-liner"></div>
        <div class="cancel-btn" @click="cancelUpload">取消上传</div>
        <div class="mask"></div>
      </div>
  ```

  ```css
  //css
      .progress-liner {
        width: 60px;
        height: 6px;
        border-radius: 6px;
        border: 1px solid #ccc;
        position: absolute;
        left: 50%;
        top: 50%;
        transform: translate(-50%, -50%);
        --progress: 20px;
  
        &::after {
          content: '';
          display: block;
          width: var(--progress);
          height: 6px;
          border-radius: 6px;
          background-color: rgb(48, 139, 219);
          position: absolute;
          left: 0;
          top: 0;
        }
  ```

  - 上传逻辑

    > 通过`input`的change事件读取到上传文件files，然后通过`FileReader`，将其转化为base64，实现图片预览的效果

    ```js
    let cnacel=null
    function selectFileChange() {
      showArea('progress')
      let files = selectFile.value?.files
      console.log(files)
      if (files?.length == 0) {
        return
      }
      const file = files![0]
      //展示预览图
      const reader = new FileReader()
      //将文件转化为base64
      reader.readAsDataURL(file)
      reader.onload = function (e) {
        if (e.target && e.target.result) {
          previewImg.value!.src = e.target.result as string
        }
      }
      //进行上传
      cancel = uploadFile(
        file,
        (progress) => {
          progressNumber.value = progress
          setProgress(progress)
        },
        (res) => {
          console.log(res)
          showArea('result')
        }
      )
    }
    
    //模拟上传接口
    function uploadFile(
      file: File,
      onProgress: (n: number) => void,
      onFinish: (flag: string, file?: File) => void
    ) {
      //模拟请求
      let p = 0
      nextTick(() => {
        setProgress(0)
      })
      const timerId = setInterval(() => {
        p++
        onProgress(p)
        if (p == 100) {
          onFinish('服务器接收到了', file)
          clearInterval(timerId)
        }
      }, 20)
      //中断请求
      return function () {
        clearInterval(timerId)
      }
    }
    ```

- 结果界面

  > 结果界面就很简单了，只需要判断中断请求函数是否生成，如果生成了就可以调用

  ```js
  //cancel
  function cancelUpload() {
    //停止上传
    cancel && cancel()
    showArea('select')
    //注意：vue里不需要让对应的上传input.value清空，因为是通过v-if进行切换的
    //每次更改都会导致其重新创建。而纯css，只能通过display:none的方式进行显示隐藏，实际上还是同一个元素
  }
  ```

  



## 拖拽上传

- 方式①：**最简单的一种**

  `注意` 

  ```
  <input type='file'/>
  ```

  **本身就支持拖拽**,所以只需要让其宽高百分百,撑满容器，同时设置其`不透明度为0`，即可实现。

  **这样甚至不需要前面那个点击外层容器，触发input的点击**

- 方式②：drag、drop

  > 首先，得让容器成为一个拖拽放置的**目标容器**(不理解的可先看下文 **文件拖拽**)

```js
function ondragover(e: DragEvent) {
  e.preventDefault()
}
function ondrop(e: DragEvent) {
  e.preventDefault()
  let files = e.dataTransfer!.files
  //注意：直接其放入到input的file中，是无法直接触发onchange事件的,
  //需要自己再次手动触发
  selectFile.value!.files = files
  selectFileChange()
}
```





# 文件拖拽

## drag、drop

- `dom`元素想要拖拽，只需要给`dom`绑定一个`draggrable`属性，就可以让这个`dom`添加上拖拽特性。

- 如果想要某个元素成为有效的`存放容器`，存放拖动元素，需要让其`作为过程容器时，阻止拖拽默认事件`，`作为结果容器时，也需要阻止默认事件`。

```js
      const box = document.querySelector('.box')
      box.ondragenter = e => {
        e.preventDefault()
      }
      box.ondragover = e => {
        e.preventDefault()
      }
      box.ondrop = e => {
        e.preventDefault()
        console.log(e.dataTransfer.files)
      }
```

**注意，img默认是可以拖拽的**

```js
<style>
  #box {
    width: 100px;
    height: 100px;
    background-color: red;
  }
</style>
​
<div id="box" draggable="true"></div>
```



### 拖拽元素的拖拽事件

`h5`对于可拖拽的元素，提供了下面三个事件用于拖拽监听

- **dragstart：** 拖拽开始
- **drag：** 拖拽中
- **dragend：** 拖拽结束

```js
      const box = document.querySelector('.box')
      box.addEventListener('dragstart', function (e) {
        console.log(e.dataTransfer)
      })
      box.addEventListener('drag', function (e) {})
      box.addEventListener('dragend', function (e) {
        console.log('拖拽结束')
      })
```



拖拽的事件对象dragEvent有以下几个属性：

![image-20231001184921852](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231001184921852.png)

常用的几个

| 属性             | 作用                                                         |
| ---------------- | ------------------------------------------------------------ |
| **target**       | 为拖拽中的元素的指向                                         |
| **clientX**      | 当前鼠标相对于浏览器窗口视口的左上角的值                     |
| **clientY**      | 当前鼠标相对于浏览器窗口视口的左上角的值                     |
| **dataTransfer** | ![image-20231001185215637](https://gitee.com/zhengdashun/pic_bed/raw/master/img/image-20231001185215637.png)<br />**files：当前拖拽的文件**<br />当一次拖拽开始发生的时候，会产生一个`DataTransfer`对象，这个`DataTransfer`对象于整个拖拽过程触发的事件共享。当这一次的拖拽行为**结束之后**，这个对象会被**销毁** |



### 过程元素的拖拽范围事件

当有个可以拖拽的元素之后，元素在拖拽的过程中，会经过一些元素，这里给他取一个形象一点的名称：过程元素

![image-20221001164233260.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3ddb3887ccf64279b9f6da389ced9b40~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

可以给这些过程元素绑定三个事件

- **dragenter：** 拖拽元素进入该`dom`时触发  
- **dragover：** 拖拽元素在该`dom`范围内触发 
- **dragleave：** 拖拽元素离开该`dom`时触发

```ini
<div class="process">过程元素</div>
```

![image-20210324124829819.png](https://gitee.com/zhengdashun/pic_bed/raw/master/img/daf30755963f4b2583a81b53cb58d522~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

```javascript
const process = document.querySelector('.process')
process.addEventListener('dragenter', function(e){  console.log('拖拽元素进入')})
process.addEventListener('dragover', function(e){  console.log('拖拽元素移动')})
process.addEventListener('dragleave', function(e){  console.log('拖拽元素离开')}) 
```

需要注意的是，这三个事件中，并不能拿到`dragStart`中设置的`DataTransfer`对象数据。(并不是`dragStart`中的同一个`dataTransfer`)

```js
      const box = document.querySelector('.box')
      const process = document.querySelector('.process')
      let t = null
      process.addEventListener('dragenter', function (e) {
        console.log(t == e.dataTransfer) //false
      })

      box.addEventListener('dragstart', function (e) {
        t = e.dataTransfer
      })
```





### 放置事件

 拖拽元素松开的位置在**目标元素中**，会触发这个函数，在这个函数中可以获取到对应的`dataTransfer`

当有可以拖拽的元素之后，总是需要有一个用于放置的地方，这个地方取一个形象的名称叫做**目标元素**。可以给目标元素绑定一个事件

- **drop：** 拖拽元素松开的位置在**目标元素中**，会触发这个函数【**在这个函数中可以拿到在`dragStart`中对应的`DataTransfer`**】
- **注意，想要触发drop事件，那么对应的`drop容器需要注册drop事件`，同时需要`阻止dropover的默认行为`**，取消其作为过程元素，成为`目标元素容器`
- **重点：`drop事件`中的 `e.dataTransfer`就是dragstart,drag.dragend中的 `e.dataTransfer` **（不是过程元素中dragover，dragenter，dargleave中的dataTransfer）



给目标元素绑定 `drop` 事件。

```javascript
javascript复制代码const target = document.querySelector('.target')
target.addEventListener('drop', function (e) {  
  console.log('松开')
})
```

尝试拖拽之后，会发现，这个函数根本没有被触发，这是因为还需要给目标元素绑定一个事件

**`dragover`为拖拽元素进入到目标元素(把它作为过程容器)的时候，触发的函数。然后在这个函数中，执行阻止默认行为，表示允许拖拽元素放置在此元素中**

这样，对应的`drop`就会触发

```javascript
javascript复制代码target.addEventListener('dragover', function (e) {  
  e.preventDefault()
})
```



**根据第三点：**

只有在`drop`中能拿到`dragStart`存放的`DataTransfer`，这个参数中，还有一个`files`字段，借助这个字段，能实现文件拖拽上传的功能。

//实现文件上传功能

```js
      result.ondragover = e => {
        e.preventDefault()
      }
      result.ondrop = e => {
        e.preventDefault()  //阻止默认打开一个新的标签页
        console.log(e.dataTransfer.files)
      }
```





### 总结：

`dragstart`和`drop`中可以获取到相同的`dataTransfer`对象，因此一般移动元素就可以在`start回调函数中在dataTransfer放置对应元素id`，然后在drop的回调中对元素进行操作，比如将其放置到drop的e.target目标元素中。

```js
    <div id="target" class="box" draggable="true"></div>
    <div class="process"></div>
    <div class="result"></div>
    <script>
      const box = document.querySelector('.box')
      box.ondragstart = function (e) {
        e.dataTransfer.effectAllowed = 'all'
        e.dataTransfer.setData('target', e.target.id)
      }
      const result = document.querySelector('.result')
      result.ondragover = function (e) {
        e.preventDefault()
      }
      result.ondrop = function (e) {
        e.preventDefault()
        let target = e.dataTransfer.getData('target')
        const moveEle = document.getElementById(target)
        const copyNode = moveEle.cloneNode()  //实现复制，拖拽直接appendChild即可
        const container = e.target
        container.appendChild(copyNode)
      }
    </script>
```

dropenter、dropover、dropend是过程元素触发，e.target也是对应的过程元素。



**实现拖拽上传文件效果**

只需要让该容器成为`放置事件的放置元素即可`。

```js
 result.ondragover = function (e) {
        e.preventDefault()
      }
      result.ondrop = function (e) {
        e.preventDefault()
        console.log(e.dataTransfer.files)
      }
```

