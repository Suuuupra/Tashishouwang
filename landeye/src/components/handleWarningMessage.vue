<template>
    <div class="handlewarningmessage" :style="{top:showlog? '10%':'-100%', zIndex:showlog?4:-10,opacity:showlog?0.95:0}">
      <el-input class='inputitem' v-model="name" placeholder="处理人姓名" />
      <el-date-picker
          class='dateitem'
          v-model="time"
          type="datetime"
          size=large
          placeholder="处理时间"
          format="YYYY-MM-DD HH:MM"
          value-format="YYYY-MM-DD HH:MM"
      >
      </el-date-picker>
      <el-input class='descriptionitem' v-model="detail" placeholder="处理情况说明"  maxlength="100" show-word-limit rows=8  type="textarea"/>
        <div class="button-container">
          <el-button color="#EBEFA5" round  >提交图片</el-button>
          <el-button color="#EBEFA5" round  @click="message" >提交日志</el-button>
          <el-button color="#EBEFA5" round  @click="$emit('close')" >关闭</el-button>
        </div>

    </div>
</template>
<script>
  import { ElMessage } from 'element-plus'
  import {ref} from "vue";

  export default{
    props:{
      showlog:{
          type: Boolean,
          default: true,
        },
      index: {
        type: Number,
        default: 0,
      },
      fetchData: {
        type: Function,
        required: true,
      }
    },
    data() {
      return {
        name: '',
        time: '',
        detail: '',
      }
    },
    methods:{
      message(){
        const data = {
          name: this.name,
          time: this.time,
          detail: this.detail,
          index: this.index,
        };
        console.log(data)

        // 发送 POST 请求到后端
        fetch('http://8.148.10.46:3050/api/HandlePost', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(data)
        })
            .then(response => {
              // 处理响应
              console.log(response);
            })
            .catch(error => {
              // 处理错误
              console.error('Error:', error);
            });
        ElMessage({
          message: '成功提交日志',
          type: 'success',
        })
        this.fetchData()
        this.$emit('close')
        }
  }}
</script>
<style>
.handlewarningmessage {
   position: fixed;
   border-radius: 10px;
   top: -100%;
   right: 0%;
   width: 350px;

   background-color: #51b679;
   display: flex;
   flex-direction: column;
   transition: opacity 0.5s ease-out;
   opacity: 0;
   z-index:10
   }
.inputitem {
   width:300px;
   margin-top: 20px;
   margin-left:20px;
   margin-bottom: 10px;
   border-radius: 10px;
   height:40px
}
.dateitem {
   width:400px;
   margin-top: 10px;
   margin-left:20px;
   margin-bottom: 10px;
   border-radius: 10px;

}
.descriptionitem {
  width:300px;
  margin-top: 10px;
  margin-left:20px;
  margin-bottom: 10px;
  border-radius: 10px;

}
.button-container {
  display: flex; 
  justify-content: space-between; 
  margin-top: 30px;
  margin-bottom: 20px;
  padding-left: 5px;
  padding-right: 5px;
}

.button-container button {
  flex: 1; 
  margin-left: 2px;
  margin-right:3px;
  font-size:18px;
  border-radius:4px;
  border:none
}
.el-button {
 font-weight:bold;
  color: #58605b;
}
.el-button:hover {
  color: #58605b;
}

</style>