<template>
	<div>
		<el-row :gutter="15">
			<!-- 视频墙 -->
			<el-col :span="19">
				<div class="meeting-container">
					<el-scrollbar height="650px" id="videoListContainer">
						<div class="video-list">
							<div class="video">
								<div class="user">
									<img class="photo" :src="mine.photo" />
									<span class="name">{{ mine.name }}{{ shareStatus ? '（共享中）' : '' }}</span>
								</div>
								<div id="localStream"></div>
							</div>
							<div class="video" v-for="one in memberList">
								<div class="user">
									<img class="photo" :src="one.photo" />
									<span class="name">{{ one.name }}</span>
								</div>
								<div :id="one.id" class="remote-stream" @dblclick="bigVideoHandle(one.id)"></div>
							</div>
						</div>
					</el-scrollbar>
					<div id="videoBig" @dblclick="smallVideoHandle()"></div>
				</div>
				<p class="desc">
					会议过程中，不需要发言的会场要主动将本地会场的MIC关闭，保证会场安静，当需要发言时要及时打开MIC。会议过程中，需要发言讨论时，先打开MIC向主会场提出请求，得到同意后再继续发言，否则请继续保持静音。发言时，要—个人一个人的发言，不要多人同时讲话，因为全向MIC会把所有人的声音混合，远端听到的声音会非常嘈杂，听不清具体说话内容。在会议进行过程中，尽量控制会场噪音，不要在会场中随意走动
				</p>
			</el-col>
			<!-- 用户列表区域 -->
			<el-col :span="5">
				<el-card shadow="never">
					<template #header>
						<div class="card-header"><span>用户列表</span></div>
					</template>
					<el-scrollbar height="555px">
						<ul class="user-list">
							<li v-for="one in userList">
								<img class="mic" src="../assets/trtc/mic.png" />
								<div class="mic-container">
									<img
										class="mic-green"
										:id="'mic-' + one.userId"
										src="../assets/trtc/mic-green.png"
									/>
								</div>
								<span>{{ one.dept }} - {{ one.name }}</span>
							</li>
						</ul>
					</el-scrollbar>
				</el-card>
				<div class="meeting-operate">
					<button :class="meetingStatus ? 'phone-btn-off' : 'phone-btn-on'" @click="phoneHandle"></button>
					<button :class="videoStatus ? 'video-btn-on' : 'video-btn-off'" @click="videoHandle"></button>
					<button :class="micStatus ? 'mic-btn-on' : 'mic-btn-off'" @click="micHandle"></button>
					<button :class="shareStatus ? 'share-btn-on' : 'share-btn-off'" @click="shareHandle"></button>
				</div>
			</el-col>
		</el-row>
	</div>
</template>

<script>
import TRTC from 'trtc-js-sdk';
import $ from 'jquery';
export default {
	data: function() {
		return {
			meetingId: null,
			uuid: null,
			appId: null,
			userSign: null,
			userId: null,
			roomId: null,
			meetingStatus: false,
			videoStatus: true,
			micStatus: true,
			shareStatus: false,
			userList: [], //进入会场的用户列表
			mine: {},
			memberList: [], //会议成员列表
			client: null,
			localStream: null,
			shareStream: null,
			stream: {}, //所有的远端流
			bigVideoUserId: null //大屏显示远端流的用户ID，切换回小屏幕的时候使用
		};
	},
	methods: {
		//用于判断当前本地使用的是本地流还是共享流，并返回相应的流对象
		getStream: function() {
		    let that = this;
		    let stream = null;
		    if (that.localStream != null) {
		        stream = that.localStream;
		    } else if (that.shareStream != null) {
		        stream = that.shareStream;
		    }
		    return stream;
		},
		phoneHandle: function() {
		    let that = this;
		    //检查浏览器是否支持在线视频会议
		    TRTC.checkSystemRequirements().then(checkResult => {
		        if (!checkResult.result) {
		            that.$alert('当前浏览器不支持在线视频会议', '提示信息', {
		                confirmButtonText: '确定'
		            });
		        } else {
		            that.meetingStatus = !that.meetingStatus;
		            if (that.meetingStatus) {
		                //记录摄像头和话筒的状态
		                that.videoStatus = true;
		                that.micStatus = true;
		                //TRTC日志输出级别为Error
		                TRTC.Logger.setLogLevel(TRTC.Logger.LogLevel.ERROR);
		
		                //生成用户签名（进入视频会才需要用户签名，不需要提前生成）
		                that.$http('meeting/searchMyUserSig', 'GET', {}, false, function(resp) {
		                    that.appId = resp.appId;
		                    that.userSig = resp.userSig;
		                    that.userId = resp.userId;
		                });
		
		                //创建TrtcClient对象
		                let client = TRTC.createClient({
		                    mode: 'rtc',
		                    sdkAppId: that.appId,
		                    userId: that.userId + '',
		                    userSig: that.userSig
		                });
		                that.client = client;
		
		                //监听新增远端流事件（用户进入会议室时候触发）
		                client.on('stream-added', event => {
		                    let remoteStream = event.stream;
		                    //订阅远端流
		                    client.subscribe(remoteStream);
		                    //从远端流获得远程用户userId(创建TrtcClient对象时候的参数)
		                    let userId = remoteStream.getUserId(); 
		                    //TODO 把新上线的用户添加到页面右侧的在线人员列表中
		                    
		                    //把远端流保存到模型层JSON中，将来大屏显示的时候要找到某个远端流停止小窗口播放，切换到大窗口播放
		                    that.stream[userId] = remoteStream;
		                });
						
						//订阅语音事件（无论本地还是远端说话，都会触发这个事件）
						client.on('audio-volume', event => {
						    event.result.forEach(({ userId, audioVolume, stream }) => {
						        //说话声音超过5，就设置话筒音量动画
						        if (audioVolume > 5) {
						            $('#mic-' + userId).css('top', `${100 - audioVolume * 3}%`);
						        } else {
						            $('#mic-' + userId).css('top', `100%`);
						        }
						    });
						});
						// 开启音量回调函数，并设置每 30ms 触发一次事件
						client.enableAudioVolumeEvaluation(30);
		                //远端流订阅成功事件
		                client.on('stream-subscribed', event => {
		                    let remoteStream = event.stream;
		                    let userId = remoteStream.getUserId();
		                    //找到视频墙中某个远端用户的格子，把其中用于显示视频的DIV，置顶覆盖用户信息
		                    $('#' + userId).css({ 'z-index': 1 }); 
		                    //在这个置顶的DIV中播放远端音视频讯号
		                    remoteStream.play(userId + ''); 
		                });
		                //订阅远端删除流事件（远端用户退出会议室）
		                client.on('stream-removed', event => {
		                    let remoteStream = event.stream;
		                    //取消订阅该远端流
		                    client.unsubscribe(remoteStream); 
		                    let userId = remoteStream.getUserId();
		                    //TODO 在页面右侧的用户列表中删除该用户
							    //根据新进入会议室的用户ID查询用户新，并且更新到上线用户列表中
							    that.$http('user/searchNameAndDept', 'POST', { id: userId }, true, function(resp) {
							        let name = resp.name;
							        let dept = resp.dept;
							        that.userList.push({ userId: userId, name: name, dept: dept });
							    });
		                    //停止播放远端流视频，并且关闭远端流
		                    remoteStream.stop();
		                    remoteStream.close();
		                    //删除模型层JSON中保存的远端流对象
		                    delete that.stream[userId];
		                    //把视频墙中该用户格子的视频DIV控件置底，显示用户基本信息
		                    $('#' + userId).css({ 'z-index': '-1' });
		                    $('#' + userId).html('');
		                });
		                
		                client
		                    .join({ roomId: that.roomId })
		                    .then(() => {
		                        //TODO 执行签到
								//执行签到
								that.$http('meeting/updateMeetingPresent','POST', { meetingId: that.meetingId }, true,function(resp) {
								    let rows = resp.rows;
								    if (rows == 1) {
								        that.$message({
								            message: '签到成功',
								            type: 'success',
								            duration: 1200
								        });
								    }
								});
		                        //成功进入会议室，然后创建本地流
		                        let localStream = TRTC.createStream({
		                            userId: that.userId + '',
		                            audio: true,
		                            video: true
		                        });
		                        that.localStream = localStream;
		                        localStream.setVideoProfile('480p'); //设置分辨率
		                        //TODO 把自己添加到上线用户列表中
								//把自己添加到上线用户列表中
								that.$http('user/searchNameAndDept', 'POST', { id: that.userId }, true, function(resp) {
								    let name = resp.name;
								    let dept = resp.dept;
								    that.userList.push({ userId: that.userId, name: name, dept: dept });
								});
		                        //初始化本地音视频流
		                        localStream
		                            .initialize()
		                            .catch(error => {
		                                console.error('初始化本地流失败 ' + error);
		                            })
		                            .then(() => {
		                                console.log('初始化本地流成功');
		                                //视频墙中第一个格子中的视频DIV置顶
		                                $('#localStream').css({ 'z-index': 1 });
		                                //播放本地音视频流
		                                localStream.play('localStream'); 
		                                //向远端用户推送本地流
		                                client
		                                    .publish(localStream)
		                                    .catch(error => {
		                                        console.error('本地流发布失败 ' + error);
		                                    })
		                                    .then(() => {
		                                        console.log('本地流发布成功');
		                                    });
		                            });
		                    })
		                    .catch(error => {
		                        console.error('进入房间失败: ' + error);
		                    });
		            } else {
		                //TODO 关闭视频会议
						//获取当前本地使用的流，有可能是本地流或者共享流
						let stream = that.getStream(); 
						that.client.unpublish(stream).then(() => {
						    // 取消发布本地流成功
						    that.client
						        .leave()
						        .then(() => {
						            console.log('成功退出会议室');
						            //关闭本地流或者共享流
						            stream.stop();
						            stream.close();
									      //清空模型层的本地流
						            that.localStream = null;
						            that.shareStream = null;
									      //清空模型层的远端流
						            that.stream = {};
									      //销毁TrtcClient对象
						            that.client = null;
						            that.userList = []; //清空用户列表
						            that.videoStatus = true;
						            that.micStatus = true;
						            that.shareStatus = false;
									      //视频墙上本地流DIV区域置底
						            $('#localStream').css({ 'z-index': '-1' });
						            $('#localStream').html('');
						            //TODO 如果是播放大屏视频的时候退出会议，退出会议后需要隐藏大屏
						        })
						        .catch(error => {
						            console.error('成功退出会议室失败' + error);
						        });
						});
		            }
		        }
		    });
		},
		created: function() {
		    let that = this;
		    //路由传入的参数
		    let params = that.$route.params;
		    that.meetingId = params.meetingId;
		    that.uuid = params.uuid;
		    let data = { meetingId: that.meetingId };
		    //查询会议所有参会人，并且生成视频墙
		    that.$http('meeting/searchOnlineMeetingMembers', 'POST', data, true, function(resp) {
		        let list = resp.list;
		        if (list != null && list.length > 0) {
		            //取出当前用户的信息
		            that.mine = list[0];
		            //其他参会人信息保存在memberList里面
		            for (let i = 1; i < list.length; i++) {
		                that.memberList.push(list[i]);
		            }
		        }
		    });
		  
		    data = {
		        uuid: that.uuid
		    };
		    // 查询视频会议室的ID
		    that.$http('meeting/searchRoomIdByUUID', 'POST', data, true, function(resp) {
		        if (resp.roomId == null) {
		            that.$message({
		                message: '不存在视频会议室',
		                type: 'error',
		                duration: 1200
		            });
		        } else {
		            that.roomId = resp.roomId;
		        }
		    });
		}
		
	},
  }
</script>

<style lang="less" scoped="scoped">
@import url('meeting_video.less');
</style>
