<!--聊天对象左侧-->
<template>
    <div style="position: relative; width: 100%; height: 100%;">
        <!--获得新增聊天对象-->
        <div class="chat-item new-add common-based-btn" @click="addSession()">
            <img src="@/assets/img/utils/plus.svg" style="width:2rem;height:2rem;" />
        </div>
        <!--新加聊天对象的弹窗-->
        <div v-if="isAddSession" class="modal-mask" @click="handleMaskClick">
            <div class="modal-container" @click.stop>
                <p class="add-title" style="margin-top: -0.5rem;">我的好友</p>
                <div v-if="friendData && friendData.length>0" style="margin-top: 0.5rem;">
                    <el-scrollbar style="height: 90%;">
                    <div v-for="(item,index) in friendData" :key=index class="flex-center-container add-item" >
                        <img :src="item.cover" class="second-common-avatar" />
                        <p style="margin: 0.2rem 0.3rem 0 0.3rem;">{{item.nickname.length>=4 ? item.nickname.substr(0,4)+"...": item.nickname}}</p>
                        <el-button class="send-based-btn common-btn-center send-btn" @click.stop="turnNewSession(item)">消息</el-button>
                    </div>
                    </el-scrollbar>
                </div>
                <div v-else>
                    <img src="@/assets/img/utils/noData.png" style="width: 2rem; height: 2rem;" />
                </div>
            </div>
        </div>
        <!--ChatGPT通道-->
        <div @click="bigModelTunnel()" class="chat-item flex-based-container"
        :class="{'chat-clicked': currentPerson.upId===0}"
        >
            <img src="@/assets/img/utils/avatar_new_01.jpg" class="first-common-avatar chat-avatar" />
            <div class="flex-column-left-max-container">
                <p style="font-weight: 600;">星火API</p>
                <p style="font-size: 0.9rem;">星火大模型伴你同行</p>
            </div>
        </div>
        <!--聊天对象列表-->
        <div v-if="personList.length===0" style="height: 99%;" class="flex-center-container">
            <p style="color: rgb(135 127 200); font-weight: 600; font-size: 1.2rem;">暂无好友~</p>
        </div>
        <div v-else v-for="(chatPerson, index) in personList" :key="index">
            <div @click="changeTunnel(chatPerson)" class="chat-item flex-based-container" 
            :class="{'chat-clicked': currentPerson.upId===chatPerson.upId}" >
                <img :src="chatPerson.avatar" class="first-common-avatar chat-avatar" />
                <div class="flex-column-left-max-container">
                    <p style="font-weight: 600;">{{chatPerson.upName.length >= 9
                        ? chatPerson.upName.substr(0, 8) + "..."
                        : chatPerson.upName}}</p>
                    <p style="font-size: 0.9rem;" :class="{
                        'me-message': chatPerson.leastMessageFrom === 1,
                        'other-message': chatPerson.leastMessageFrom === 2
                    }"
                    >{{chatPerson.leastMessage.length >= 10 ? 
                    chatPerson.leastMessage.substr(0, 9) + "..." : chatPerson.leastMessage}}</p>
                </div>
            </div>
        </div>
        <!--后续数据-->
        <div class="chat-item get-more common-based-btn">
            <img src="@/assets/img/utils/down.svg"  style="width:2rem;height:2rem;" />
        </div>
    </div>
</template>

<script setup>
import { onMounted, onBeforeMount, ref, inject, defineProps } from 'vue'
import { fetchAllPeople, addNewChat, changeSessionTime } from "@/api/chat"
import { fetchFollowingsList, fetchFollowersList } from "@/api/user"
import { useUserInfo } from "@/store/userInfo"
import { useChat } from "@/store/chat"
import { useRoute } from "vue-router"
const isAddSession = ref(false) // 是否添加会话
const route = useRoute()
const chatSession = useChat() // 使用聊天信息
const userInfo = useUserInfo() // 使用登录信息 
const userId = userInfo.getId() // 登录用户的id
const personList = ref([]) // 聊天对象列表
const friendData = ref([]) // 好友列表
const defaultBigModel = { // 默认是大模型的id
    upId: 0, // 聊天对象的id
    upName: "星火API", // 聊天对象的name
    intro: "星火大模型伴你同行",
    avatar: require("@/assets/img/avater.png"), // 聊天对象的头像
    leastMessage: "星火大模型伴你同行",
    leastMessageFrom: 2, // 0是不存在，1是自己，2是对方
    updatedUnreadFlag: true, // 是否有新的未读消息
    unreadNum: 1, // 未读新消息的条数 
}
const currentPerson = ref(defaultBigModel) // 当前用户
// 修改用户
// const updateChat = inject("updateChat")
// 切到星火大模型
const bigModelTunnel = () => {
    currentPerson.value = defaultBigModel
    chatSession.setCurrentUp(0, defaultBigModel.upName)
    updateChat({
        upId: 0, // 聊天对象的id
        upName: "星火API", // 聊天对象的name
        intro: "星火大模型伴你同行",
        avatar: require("@/assets/img/avater.png"), // 聊天对象的头像
        leastMessage: "星火大模型伴你同行",
        leastMessageFrom: 0, // 0是不存在，1是自己，2是对方
        updatedUnreadFlag: false, // 是否有新的未读消息
        unreadNum: 0, // 未读新消息的条数 
    })
}
// 新增会话按钮
const addSession = async() => {
    isAddSession.value = true 
    const followingArray = await fetchFollowingsList(userId)
    const followerArray = await fetchFollowersList(userId)
    const mergedArray = followingArray.concat(followerArray)
    // friendData.value = Array.from(new Set(mergedArray))
    friendData.value = [...new Set(mergedArray)]
    console.log(`获取的好友${JSON.stringify(friendData.value)}`)
}
const isNeedNew = (upId) => {
    return personList.value.some(person=>person.upId !== upId)
}
const turnNewSession = async(upInfo) => { // 进入新会话
    if(upInfo.userId===0 || !upInfo.nickname) {
        ElMessage.error("传参错误")
        return 
    }
    if(isNeedNew(upInfo.userId)) { // 如果数字upId与personList数组中的任一元素upId中不符，则新建
        personList.value.push({
            avatar: upInfo.cover,
			createTime: new Date(),
			upName: upInfo.nickname,
			upId: upInfo.userId,
            leastMessage: "",
            leastMessageFrom: 0
        })      
        await addNewChat(userId, upInfo.userId)
        await changeSessionTime(userId, upInfo.userId, "")
    }
    isAddSession.value = false 
    currentPerson.value.upId = upInfo.userId
    chatSession.setCurrentUp(upInfo.userId, upInfo.nickname)
}
// 获取分享者的id
const props = defineProps({
    receiverId: {
        type: Number,
        required: false,
        default: null
    }
})
// 修改当前用户的Tunnel
const changeTunnel = (Tunnel) => {
    if(Tunnel.upId!==currentPerson.upId) {
        currentPerson.value = Tunnel // 当前对象的新值
        chatSession.setCurrentUp(Tunnel.upId, Tunnel.upName) //  
        updateChat(Tunnel)
    }
}
// 点击蒙版外的会退出弹窗
const handleMaskClick = (event) => {
    if (!event.target.classList.contains('modal-container')) {
        isAddSession.value = false
    }
}
onMounted(async()=>{
    // 获得聊天对象列表
    personList.value = await fetchAllPeople(userId)
    if(personList.value.length>0) {
        currentPerson.value = personList.value[0]
        if(!chatSession.getUpId() || chatSession.getUpId()===0) {
            chatSession.setUpId(currentPerson.value.upId)
        }
        if(!chatSession.getUpName() || chatSession.getUpName()==='') {
            chatSession.setUpName(currentPerson.value.upName)
        }
    }
    // 假如传来了当前聊天对象
    if (route &&route.query && route.query.receiverId) { // NOTE 使用route.query?没用，需要显式检查
        // 如果不存在，则新建

    } 
})
</script>

<style lang="scss" scoped>
@import "@/assets/css/messagePage.scss";
@mixin func-btn {
    border-radius: 20px;
    &:hover,&:active {
        border-radius: 20px;
    }
}
.me-message {
    color: #4e6ba5;
}
.other-message {
    color: #0a725e;
}
.chat-avatar {
    margin: 0 1.3rem 0 1.2rem;
}
.chat-item {
    margin: 0.5rem 0 0.5rem 0.5rem;
    width: 91%;
    padding: 0.5rem;
    cursor: pointer;
    &:active {
        background: #d3def5;
        border-radius: 10px;
    }
}
.new-add {
    @include func-btn;
    background: #a2b7e1;
    &:hover,&:active {
        background: #4e6ba5;
    }
}
.get-more {
    @include func-btn;
    border: 1px solid #a2b7e1;
    position: absolute;
    bottom: 0.5rem;
    &:hover, &:active {
        background: #fff;
    }    
}
.chat-clicked {
    background: #d3def5;
    margin: 0.5rem 0 0.5rem 0.5rem;
    border-radius: 10px;
}
.modal-mask {
    position: absolute;
    top: -0.5rem;
    left: 0;
    width: 103%;
    height: 101%;
    background-color: rgba(0, 0, 0, 0.5); /* 半透明黑色背景 */
    z-index: 999; /* 置于其他内容之上 */
    display: flex;
    justify-content: center;
    align-items: center;
    border-radius: 20px;
}
.modal-container {
  background-color: white;
  padding: 20px;
  border-radius: 15px;
  min-height: 3rem;
  max-height: 20rem;
  width: 70%;
  .add-item {
    width: 100%;
    align-items: stretch;
    margin-bottom: 0.5rem;
  }
}
.add-title {
    font-weight: 600;
    font-size: 1.1rem;
}
@media screen and (min-width: 1020px) {
    .get-more {
        bottom: 1rem;
    }
    .chat-item {
        width: 93%;
    }
}
.send-btn {
    width: auto;
    height: auto;
    font-size: 0.9rem;
    padding: 0.1rem 0.2rem;
    color: #fff;
}
</style>