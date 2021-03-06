<template>
  <div class="main">
    <div class="header">
      <div>
        <el-image class="img" :key="getImgSrc(photo.fileId)" fit="cover" :src="getImgSrc(photo.fileId)" lazy />
      </div>
      <div class="info">
        <span>{{photo.name}} (<strong>{{photo.imgCount}}</strong>) </span>
        <span>
          <editorImage pickeName="上传图片" color="#1890ff" :accept="accept" class="editor-upload-btn" @successCBK="imageSuccessCBK" />
        </span>
      </div>
    </div>
    <div class="content">
      <el-scrollbar style='height:100%'>
        <el-row class="row">
          <el-col class="col" v-for="(file,index) in fileList" :key="file.fileId" :xs="12" :sm="6" :md="4" :xl="1">
              <div class="gutter">
                <el-image @contextmenu.prevent.stop="contextmenu(file.id,index,$event)" v-longtap.stop="{fn:openContext, fileId:file.id, index:index}"  :preview-src-list="priviewSrcList" :key="getImgSrc(file.fileId)" fit="cover" :src="getImgSrc(file.fileId)" />
                <div class="descr">
                  <span contenteditable="true" @blur="changeFileName(file,index,$event)">{{ file.name }}</span>
                  <span >{{file.updateTime | excludSec}}</span>
                </div> 
              </div>
          </el-col>
        </el-row>
      </el-scrollbar>
    </div>
  </div>
</template>

<script>
import { log } from 'util';
import focusOnCondition from '@/directive/focus';
import { getFileList, batchAddPhoto, deletePhoto, updatePhoto} from '@/api/photo';
import { downloadFile} from '@/api/file';
import { MessageBox } from 'element-ui';
import editorImage from '@/components/Tinymce/components/EditorImage'
document.oncontextmenu = function() { return false; }
export default {
  directives: { focusOnCondition },
  components: { editorImage },
  data() {
    return {
        visible: false,
        searchFileName: '',
        fileList: [],
        priviewSrcList:[],
        file: null,
        tempFilelist: [],
        pid:'',
        photo:{},
        fileId:'',
        defaultImgPath: require('@/assets/img/default.jpg'),
        accept: '.jpg,.jpeg,.png,.gif,.bmp,.JPG,.JPEG,.PBG,.GIF,.BMP'
    };
  },
  watch: {
    fileList: function(newFileList, oldFileList) {
      this.photo.imgCount = newFileList.length;
      this.photo['fileId'] = newFileList[0].fileId;
      var that = this;
      newFileList.forEach(function(item, index) {
          that.priviewSrcList.push(that.getImgSrc(item.fileId));
      })
    }
  },
  mounted: function() {
    this.pid = this.$route.params.pid;
    this.freshFileList({ pid: this.pid,hasOwer:'y' });
  },
  methods: {
    imageSuccessCBK(arr) {
      arr.forEach(v => {
        if(v.hasSuccess){
          this.tempFilelist.push({fileId:v.uid,pid:this.pid,name:v.fileName});
        }
      });
      batchAddPhoto(this.tempFilelist).then(res => {
        this.$message.success('保存成功');
        this.fileList = this.fileList.concat(res.data);
        this.tempFilelist = [];
      })
    },
    getImgSrc(key) {
      if (!key) {
        return this.defaultImgPath;
      }
      return this.$store.state.settings.serverPath + 'sysFile/priviewImg?fileKey=' + key;
    },
    freshFileList(data) {
      const loading = this.$loading({
          lock: true,
          text: '加载中',
          spinner: 'el-icon-loading',
          customClass: 'loading',
          target: '.main'
      });
      this.fileList = [];
      getFileList(data).then((res) => {
        const list = res.data.list;
        this.fileList = list;
        this.photo = res.data.photo;
      })
      loading.close();
    },
    changeFileName(file,index, event) {
      const changeName = event.target.innerText;
      if(file.name == changeName){
        return ;
      }
      file.name = changeName;
      updatePhoto(file).then(res => {
        this.$message.success('修改成功');
        if (this.fileList[index].fileName !== changeName) {
          this.fileList[index].fileName = changeName;
        }
      });
    },
    deleteFiles(id,index) {
      this.$confirm('此操作将永久删除该文件, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
          center: true
        }).then(() => {
           deletePhoto({id:id}).then(res => {
            this.fileList.splice(index, 1);
            this.$message.success('删除成功！');
          })
        })
    },
    downLoad(id,index){
        downloadFile( this.fileList[index].fileId);
    },
    openContext(ev,value){
      this.contextmenu(value.fileId,value.index, ev);
    },
    contextmenu(id,index, event) {
      this.$contextmenu({
        items: [
          { label: '查看', icon: 'el-icon-back', divided: true, onClick: () => {
            event.target.click();
          } },
          { label: '下载', icon: 'el-icon-download', divided: true , onClick: () => {
            this.downLoad(id,index);
          } },
          // { label: '移动到', divided: true },
          { label: '删除', divided: true, icon: 'el-icon-delete', onClick: () => {
            this.deleteFiles(id,index);
          } },
          { label: '重命名', divided: true, onClick: () => {
            event.target.parentNode.nextElementSibling.children[0].focus();
          } },
        ],
        event,
        zIndex: 3,
        minWidth: 100
      });
      return false;
    },

    searchFile() {
      if (this.searchFileName == '' && this.tempFilelist.length > 0) {
        this.freshFileList({ pid: this.rootId,searchName:this.searchFileName });
      }
    },
    back(e) {
      if (!this.leftArrowEnable) {
        return false;
      }
      this.searchFileName = '';
      if (this.tempFilelist.length > 0) {
        this.fileList = this.tempFilelist;
        this.tempFilelist = [];
        if (this.currentIndex == 0) {
          this.leftArrowEnable = false;
          return ;
        }
      }
      this.currentIndex--;
      if (this.currentIndex == 0) {
        this.leftArrowEnable = false;
      }
      this.rightArrowEnable = true;
      this.currentPath.splice(this.currentPath.length - 2, 2);
    },
    forward(e) {
      if (!this.rightArrowEnable) {
        return false;
      }
      this.leftArrowEnable = true;
      this.currentIndex++
      if ((this.currentIndex + 1) == this.pathIds.length) {
        this.rightArrowEnable = false;
      }
    },
    enterFloder(file) {
      if (file.ended) {
        return false;
      }
      this.leftArrowEnable = true;
      this.pathIds.push(file.fileKey);
      this.currentIndex++;
      this.currentPath.push('<a class="path-a" data-id="' + file.fileKey + '">' + file.fileName + '</a>');
      this.currentPath.push(this.arrow);
    },
    focusDiv(index) {
      this.number = index;
    }
  }
};
</script>
<style lang="scss">
.search .el-input__inner{
    padding-left: 0px ;
    padding-right: 10px ;
    border: none ;
    width: 220px ;
}
.search i{
  font-size: 16px;
}
.el{
  color:#F8C92A
}
a.path-a{
  font-size: 14px !important;
  vertical-align: top;
}
a.path-a:hover{
  color:#4DB3FF !important;
};
.el-icon-back{
  transform: rotate(135deg);
}
.dialog{
  .el-dialog__header{
    height: 20px;
  }
  ul{
    padding: 10px 10px 20px 10px;
    margin: 0px;
  }
  li:first-child{
    height: 60px;
    border-bottom: 1px dashed #ccc;
    line-height: 60px;
  }
  li{
    height: 40px;
    line-height: 40px;
    list-style: none;
    span{
        display: inline-block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
        width: 300px;
    }
    span:first-child{
      width: 80px;
      color: #aaa;
    }
    span + span{
          width: 280px;
    }
    div{
      display: inline-block;
      *{
        vertical-align: middle;
      }
      i{
          font-size: 30px !important;
      }
      span{
        display: inline-block;

        margin: 0px 10px;
      }

    }
  }
}
.el-scrollbar__wrap{
  overflow-x:hidden;
}
</style>
<style lang="scss" scoped>
.main {
  height: 100%;
  color: #666;
  .footer{
    letter-spacing: 1px;
    height: 30px;
    line-height: 30px;
    width: 100%;
    border-top: 1px solid #ccc;
    div{
      display: inline-block;
      margin: 0px 10px;
      span{
        letter-spacing: 2px;
      }
    }
    div+div{
      margin-left: 20px;
    }
  }
  .header{
    font-size: 16px;
    height: 90px;
    width: 100%;
    border-bottom: 1px solid #ccc;
    > div{
      display: inline-block;
      vertical-align: middle;
      height: 100%;
      span{
        display: block;
        height: 45px;
      }
      span:nth-child(1){
        color: #999;
        line-height: 50px;
      }
      span:nth-child(2){
        line-height: 40px;
        color: #aaa;
      }
    };
    .img{
      width: 60px !important;
      height: 65px;
      margin: 15px 15px 0px 15px;
    };
    .operate{
      margin-left:8px;
      span{
        cursor: pointer;
      };
      span:last-child{
        margin-left: 16px;
      };
      span.disabled{
        color: #bbb
      };
      
    };
    .path{
      width: 70%;
      width: -moz-calc(100% - 420px);
      width: -webkit-calc(100% - 420px);
      width: calc(100% - 420px);
      margin-left: 10px;
      span{
        width: 100%;
        text-overflow: ellipsis;
        overflow: hidden;
        white-space: nowrap;
        display: block;
        i.el-icon-arrow-right{
          font-size: 14px;
        };

      };

    };
    .separate{
      border-left: 1px solid #ccc;
      height: 20px;
      vertical-align: middle;
      margin-left: 20px;
      margin-right: 10px;
    };
  };
  .content{
    height: calc(100vh - 175px);
    overflow: hidden;
    background: #e8f4ff;
    .row{
      .col{
        position: relative;
        text-align: center;
        display: inline-block;
        padding: 10px;
        vertical-align: top;
        > * {
          box-sizing: border-box;
          display: block;
        };
        .gutter{
          padding: 10px;
          background-color: #fff;
          i{
            margin: 0 auto;
            font-size: 80px;
          };
          .descr{
            margin-top: 10px;
            span{
              white-space: nowrap;
              overflow: hidden;
              text-overflow: ellipsis;
              width: 100%;
              text-align: left;
              font-size: 12px;
              display: block;
              padding-left: 2px;
            };
            span:nth-child(1){
              border-top: 1px solid #eee;
              height: 40px;
              line-height: 40px;
              color:#333;
            };
            span:nth-child(2){
              height: 20px;
              line-height: 20px;
              color: #aaa;
            }
          }
        }
      }
    }
  }
}

@media screen and (max-width: 300px) {
    body {
        background-color:lightblue;
    }
}

</style>
