<template>
  <div class="upload">
    <input type="file" @change="handle($event)" name="model" accept="image/*">
    <i class="iconfont icon-bl_icons_camera"></i>
    <img :src="imgSrc" alt="" v-show="imgSrc" :name="model">
  </div>
</template>
<script>
import EXIF from 'exif-js'
  export default{
    data(){
      return {
        imgSrc:''
      }
    },
    props:[
      'model'


    ],
    created(){

    },
    ready(){
      

    },
    methods:{
      handle(evt){
        var _name = this.model
        const files = Array.prototype.slice.call(evt.target.files);

        let that = this


        files.forEach(function (file, i) {
            var orientation;
            if (!/\/(?:jpeg|png|gif)/i.test(file.type)) return;
            //读取图片的元信息
            EXIF.getData(file,function(){
                orientation=EXIF.getTag(this,'Orientation');
            });

            let reader = new FileReader();

            reader.onload = function () {
                let result = this.result;
                that.imgSrc = result;
                //使用exif
                that.getImgData(this.result,orientation,function(data){
                    //这里可以使用校正后的图片data了
                    var img = new Image()
                    img.src = data


                    //图片加载完毕之后进行压缩，然后上传
                    if (img.complete) {
                      callback();
                    } else {
                      img.onload = callback;
                    }


                    function callback() {
                        var data = that.compress(img);
                        that.upload(data, file.type, file.name, _name);


                    }
                });







            };

            reader.readAsDataURL(file);
        })


      },
      //压缩图片
      compress(img){
          //用于压缩图片的canvas
          let canvas = document.createElement("canvas");
          let ctx = canvas.getContext('2d');

          //    瓦片canvas
          var tCanvas = document.createElement("canvas");
          var tctx = tCanvas.getContext("2d");

          let initSize = img.src.length;
          let width = document.querySelector(".upload").offsetWidth;
          let height = document.querySelector(".upload").offsetHeight;

          //如果图片大于四百万像素，计算压缩比并将大小压至400万以下
          var ratio;
          if ((ratio = width * height / 4000000) > 1) {
            ratio = Math.sqrt(ratio);
            width /= ratio;
            height /= ratio;
          } else {
            ratio = 1;
          }
          canvas.width = width * 2;
          canvas.height = height * 2;
          //铺底色
          ctx.fillStyle = "#fff";
          ctx.fillRect(0, 0, canvas.width, canvas.height);
          //如果图片像素大于100万则使用瓦片绘制
          var count;
          if ((count = width * height / 1000000) > 1) {
            count = ~~(Math.sqrt(count) + 1); //计算要分成多少块瓦片
              //计算每块瓦片的宽和高
            var nw = ~~(width / count);
            var nh = ~~(height / count);
            tCanvas.width = nw;
            tCanvas.height = nh;
            for (var i = 0; i < count; i++) {
              for (var j = 0; j < count; j++) {


                tctx.drawImage(img, i * nw * ratio, j * nh * ratio, nw * ratio * 2, nh * ratio * 2, 0, 0, nw, nh);
                ctx.drawImage(tCanvas, i * nw, j * nh, nw * 2, nh * 2);


              }
            }
          } else {
            ctx.drawImage(img, 0, 0, width * 2, height * 2);

          }


          //进行最小压缩
          let ndata = canvas.toDataURL('image/jpeg', 0.9);

          console.log('压缩前：' + initSize);
          console.log('压缩后：' + ndata.length);
          console.log('压缩率：' + ~~(100 * (initSize - ndata.length) / initSize) + "%");


          return ndata;
      },
      //上传图片
      upload(basestr, type, name, model){
        let text = window.atob(basestr.split(",")[1]);
        let buffer = new ArrayBuffer(text.length);
        let ubuffer = new Uint8Array(buffer);

        for (let i = 0; i < text.length; i++) {
            ubuffer[i] = text.charCodeAt(i);
        }

        let Builder = window.WebKitBlobBuilder || window.MozBlobBuilder;
        let blob;

        if (Builder) {
            let builder = new Builder();
            builder.append(buffer);
            blob = builder.getBlob(type);
        } else {
            blob = new window.Blob([buffer], {type: type});
        }



        let formdata = new FormData();
        formdata.append(model, blob, name);
        var post = {
          params: formdata
        }
        //选择完毕触发事件
        this.$emit('select-complete', post);

      },
      getImgData(img,dir,next){
        // @param {string} img 图片的base64
        // @param {int} dir exif获取的方向信息
        // @param {function} next 回调方法，返回校正方向后的base64
          var image=new Image();
          image.onload=function(){
            var degree=0,drawWidth,drawHeight,width,height;
            drawWidth=this.naturalWidth;
            drawHeight=this.naturalHeight;
            //以下改变一下图片大小
            var maxSide = Math.max(drawWidth, drawHeight);
            if (maxSide > 1024) {
                var minSide = Math.min(drawWidth, drawHeight);
                minSide = minSide / maxSide * 1024;
                maxSide = 1024;
                if (drawWidth > drawHeight) {
                    drawWidth = maxSide;
                    drawHeight = minSide;
                } else {
                    drawWidth = minSide;
                    drawHeight = maxSide;
                }
            }
            var canvas=document.createElement('canvas');
            canvas.width=width=drawWidth;
            canvas.height=height=drawHeight;
            var context=canvas.getContext('2d');
            //判断图片方向，重置canvas大小，确定旋转角度，iphone默认的是home键在右方的横屏拍摄方式
            switch(dir){
               //iphone横屏拍摄，此时home键在左侧
                case 3:
                    degree=180;
                    drawWidth=-width;
                    drawHeight=-height;
                    break;
                //iphone竖屏拍摄，此时home键在下方(正常拿手机的方向)
                case 6:
                    canvas.width=height;
                    canvas.height=width;
                    degree=90;
                    drawWidth=width;
                    drawHeight=-height;
                    break;
                //iphone竖屏拍摄，此时home键在上方
                case 8:
                    canvas.width=height;
                    canvas.height=width;
                    degree=270;
                    drawWidth=-width;
                    drawHeight=height;
                    break;
            }
            //使用canvas旋转校正
            context.rotate(degree*Math.PI/180);
            context.drawImage(this,0,0,drawWidth,drawHeight);
            //返回校正图片
            next(canvas.toDataURL("image/jpeg",.4));
         }
          image.src=img;
        }


    }

  }
</script>
<style scoped>
  .upload{
    display: flex;
    justify-content: center;
    align-items: center;
    width: 200px;
    height: 250px;
    position: relative;
    border: 3px solid #dab874;
    box-sizing: border-box;
    z-index: 8;
    
  }
  img,input{
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
    i{
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%,-50%);
      font-size: 0.5rem;
      color: #dab874;
    }
    input{
      opacity: 0;
      z-index: 99;
    }
</style>
