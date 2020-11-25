### 背景
最近有一个很简单的需求，实现一个拖拽的按钮，并且自动释放的时候会贴合在左右的边上

### 状态
1. 拖拽完成状态
2. 拖拽中状态

一般来说我做一些需求的时候，都会先梳理好会有几个状态，并且不同的action会导致有不同的状态；

### 实现思路
1. 拖拽之前
一开始是贴着边上的
2. 拖拽中的时候，将按钮跟随者手指移动
3. 拖拽结束之后，进行判断是贴着左边还是右边

### 核心代码实现
```JS
import './index.scss'; // 样式文件
import { Vue, Component, Prop, Provide } from 'vue-property-decorator';
import debounce from '@/utils/debounce'; // 这里是一个防抖函数

@Component
export default class ABtn extends Vue {
  @Provide() isShow: boolean = true;
  @Provide() isDown: boolean = false;
  private whichSide: string = 'right'; // 默认是右边，然后根据释放的地方判断是左边还是右边
  private position: any = {x: 0, y: 0};
  private nx: any = '';
  private ny: any = '';
  private dx: any = '';
  private dy: any = '';
  private xPum: any = '';
  private yPum: any = '';
  @Provide() type: string = 'unfold'; // 分为unfold，folded，moving 三个状态
  @Prop({ required: true, type: Function })
  private onOpen!: () => void;
  @Prop({ default: '' }) private AUrl!: string;
  @Prop({ default: null }) private AId!: string;
  @Prop({ default: null }) private AName!: string;

  // 这里是按钮中还有一些事件
  public handleClick(): void {
    console.log('handleClick.....');
    // 如果是 folded 状态下
    if (this.type === 'folded') {
      this.type = 'unfold';
      this.autoSetTypeToFolded();
    } else if ( this.type === 'unfold') {
      window.open(`${AShareUrl}?AId=${this.AId}`);
    } else {
      console.log('移动中状态');
    }
  }

  public handleClickSubScribe(): void {
    // handleClickSubScribe 处理点击的事情
  }

  private handleDown(event: any): void {
    console.log('handleDown: ', event);
    if (this.type === 'unfold') {
      return;
    }
    const moveDiv = document.getElementById('btnId'); // 根据id获取DOM元素
    let touch;
    if (event.touches) {
      touch = event.touches[0];
    } else {
      touch = event;
    }
    this.position.x = touch.clientX;
    this.position.y = touch.clientY;
    this.dx = moveDiv!.offsetLeft; // ts会提示moveDiv可能为空，此时要么判空，要么还可通过变量后添加 ! 操作符告诉 TypeScript 该变量此时非空
    this.dy = moveDiv!.offsetTop;
    if (!this.isDown) {
      this.isDown = true;
      this.type = 'moving';
    }
  }

  private handleEnd(): void {
    console.log('handleEnd');
    const moveDiv = document.getElementById('btnId');
    if (this.jugeWhichSide() === 'right') {
      moveDiv!.style.right = '0px';
      moveDiv!.style.left = 'auto';
    } else {
      moveDiv!.style.left = '0px';
      moveDiv!.style.right = 'auto';
    }

    if (this.isDown) {
      this.isDown = false;
      this.type = 'folded';
    }
  }

  private handleMove(e: any): void {
    console.log('handleMove...');
    e.stopPropagation(); // 阻止浏览器冒泡和事件捕捉
    e.preventDefault(); // 组织浏览器默认行为：上下滑动时空白，长按选中文字和图片
    const moveDiv = document.getElementById('btnId');
    if (this.isDown) {
      let touch;
      if (e.touches) {
        touch = e.touches[0];
      } else {
        touch = e;
      }
      this.nx = touch.clientX - this.position.x;
      this.ny = touch.clientY - this.position.y;
      // 不允许超出边框
      this.xPum = this.dx + this.nx;
      if ( this.xPum < moveDiv!.offsetWidth ) { this.xPum = moveDiv!.offsetWidth; }
      if ( this.xPum > window.innerWidth ) { this.xPum = window.innerWidth; }
      this.yPum = this.dy + this.ny;
      if ( this.yPum < 0 ) {
        this.yPum = 0;
      }
      if ( this.yPum > (window.innerHeight - moveDiv!.offsetHeight) ) {
        this.yPum = window.innerHeight - moveDiv!.offsetHeight;
      }

      moveDiv!.style.left = 'auto';
      moveDiv!.style.right = (window.innerWidth - this.xPum) + 'px';
      moveDiv!.style.top = this.yPum + 'px';
    }
  }

  // 公共方法们
  private autoSetTypeToFolded(): void {
    debounce(() => {
      this.type = 'folded';
    } , 3000)();
  }

  private jugeWhichSide(): string {
    const moveDiv = document.getElementById('btnId');
    const xcenter: any = ( (moveDiv as any).offsetLeft ) + ( (moveDiv as any).offsetWidth ) / 2;
    if ( (xcenter as any) > window.innerWidth / 2 ) {
      this.whichSide = 'right';
      return 'right';
    } else {
      this.whichSide = 'left';
      return 'left';
    }
  }

  // 生命周期
  created(): void {
    console.log('created....');
    this.autoSetTypeToFolded();
  }

  mounted(): void {
    console.log('mounted....');
    document.addEventListener('mousemove', this.handleMove); // 还必须绑定在document上
    // 为提高性能，chrome56之后， window docuemnt body 触屏事件默认无法取消默认事件， 需要给 addEventListener() 指定第三个参数
    document.addEventListener('touchmove', this.handleMove, { passive: false });
    document.addEventListener('touchcancel', this.handleEnd); // 如果是因为意外取消的，也需要处理
  }

  destroyed(): void {
    document.removeEventListener('mousemove', this.handleMove);
    document.removeEventListener('touchmove', this.handleMove);
    document.removeEventListener('touchcancel', this.handleEnd);
  }

  render() {

      const openABtnWraperClass = this.whichSide === 'right' ?
      'btn-wraper-right' : 'btn-wraper-left';
      const OpenAWraperClass = this.type === 'moving' ?
      openABtnWraperClass + ' btn-wraper btn-wraper-moving' :
      openABtnWraperClass + ' btn-wraper';

      return (
        <div>
          { this.isShow && this.whichSide === 'right' ? (
            <div class = { OpenAWraperClass }
              id = "btnId"
              onMouseDown = { this.handleDown }
              onTouchstart = { this.handleDown }
              onMouseMove = { this.handleMove }
              onTouchMove = { this.handleMove }
              onMouseUp = { this.handleEnd }
              onTouchend = { this.handleEnd }
            >
                <img class = "img"
                src = {this.AUrl}
                alt = "btn"
                onClick = {this.handleClick}
                />
                { this.type === 'unfold' ? <span class="name"
                  onClick = {this.handleClick}
          >{ this.AName }</span> : ''}
                { this.type === 'unfold' ? <span class="subscribe"
                  onClick = {this.handleClickSubScribe}
                >+订阅</span> : '' }
            </div>
            ) :  this.isShow && this.whichSide === 'left' ?  (
              <div class = { OpenAWraperClass }
                id = "btnId"
                onMouseDown = { this.handleDown }
                onTouchstart = { this.handleDown }
                onMouseMove = { this.handleMove }
                onTouchMove = { this.handleMove }
                onMouseUp = { this.handleEnd }
                onTouchend = { this.handleEnd }
              >
                  { this.type === 'unfold' ? <span class="subscribe"
                    onClick = {this.handleClickSubScribe}
                  >+订阅</span> : '' }
                  { this.type === 'unfold' ? <span class="name"
                    onClick = {this.handleClick}
                  >{ this.AName }</span> : ''}
                  <img class = "img"
                  src = {this.AUrl}
                  alt = "btn"
                  onClick = {this.handleClick}
                  />
              </div>) : ''}
        </div>

      );
  }
}


```