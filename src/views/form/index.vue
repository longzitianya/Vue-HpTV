<template>

  <div class="app-container" style="width: 700px">
    <el-form ref="form" :model="form" label-width="120px">
      <el-form-item label="课程名称">
        <el-input v-model="form.name" :disabled="true"/>
      </el-form-item>
      <el-form-item label="授课教师账户">
        <el-input v-model="form.teacherUser" :disabled="true"/>
      </el-form-item>
      <el-form-item label="课程预览">
        <el-image
          v-show="ifexit"
          style="width: 200px; height: 200px"
          :src="url"
          :preview-src-list="srcList">
        </el-image>
      </el-form-item>
      <el-form-item label="课程编码">
        <el-input
          style="width: 100px"
          v-model="form.liveType"
          :disabled="true">
        </el-input>
      </el-form-item>

      <el-form-item label="金额">
        <el-input v-model="form.amount" style="width: 100px" :disabled="true"/>
      </el-form-item>
      <el-form-item label="剩余时间">
        <el-time-select
          v-model="value"
          :readonly="true"
        >
        </el-time-select>
      </el-form-item>
      <el-form-item label="剩余名额">
        <el-progress class="volume" :text-inside="true" :stroke-width="22" :percentage="percentage"
                     status="warning"></el-progress>
      </el-form-item>

      <el-form-item>
        <el-button type="primary" @click="onSubmit" :disabled="this.isEnd">{{submit}}</el-button>
        <!--        <el-button @click="onCancel">Cancel</el-button>-->
      </el-form-item>
    </el-form>
  </div>
</template>

<script>

    import {showGoods, createOrder, activeFinish} from "../../api/goods";
    import {losePay} from "../../api/order";
    import {payOrder} from "../../api/order";

    export default {

        data() {
            return {
                form: {
                    name: '',
                    resource: '',
                    desc: '',
                    amount: '',
                    teacherUser: '',
                    liveType:''
                },
                ifexit: false,
                value: '',
                isEnd: false,//倒计时是否结束
                endTime: new Date(),//应为接口获取到的结束时间
                submit: '立即抢购',
                url: '',
                srcList: [
                    {url: '',}
                ],
                percentage: 0,
                alipayWap:''
            }

        },
        created() {
            this.show();
        },
        methods: {
            openPay(formdata) {
                this.$confirm('您是否已经支付订单', '提示', {
                    confirmButtonText: '确认支付',
                    cancelButtonText: '取消订单',
                    type: 'warning'
                }).then(() => {
                    createOrder(formdata).then((request) => {
                        console.log(request);
                        if (request.state === 1) {
                            this.$message({
                                message: '恭喜你，成功抢到！',
                                type: 'success'
                            });
                            this.isEnd = true
                        } else {
                            this.$message(request.message);
                        }
                    })
                }).catch(() => {
                    losePay();
                });

            },
            onSubmit() {
                let formdata = new FormData();
                formdata.append("username", sessionStorage.getItem("username"));
                formdata.append("courseName", this.form.name);
                formdata.append("total_amount", this.form.amount);
                let outTradeNo = "";  //订单号
                for (var i = 0; i < 6; i++) //6位随机数，用以加在时间戳后面。
                {
                    outTradeNo += Math.floor(Math.random() * 10);
                }
                outTradeNo = new Date().getTime() + outTradeNo;  //时间戳，用来生成订单号。
                formdata.append("out_trade_no",outTradeNo);
                formdata.append("teacherUser",this.form.teacherUser);
                formdata.append("liveType",this.form.liveType);
                let AlipayInfo = {
                    "out_trade_no" : outTradeNo,
                    "subject" : this.form.name,
                    "total_amount" : this.form.amount,
                };
                const newTab = window.open();
                payOrder(AlipayInfo).then( (respond) => {
                    if (respond.state === 0) {
                        this.$message(respond.message);
                    }
                    this.alipayWap = respond;
                    const div = document.createElement('div');
                    div.innerHTML = respond.message; //此处form就是后台返回接收到的数据
                    newTab.document.body.appendChild(div);
                    newTab.document.forms[0].submit();
                    this.openPay(formdata)
                });
                // createOrder(formdata).then((request) => {
                //     console.log(request);
                //     if (request.state === 1) {
                //
                //         this.$message({
                //             message: '恭喜你，成功抢到！',
                //             type: 'success'
                //         });
                //         this.isEnd = true
                //     } else {
                //         this.$message(request.message);
                //     }
                // })

            },
            onCancel() {
                this.$message({
                    message: 'cancel!',
                    type: 'warning'
                })
            },
            countdown() {
                // 目标日期时间戳
                const end = Date.parse(new Date(this.endTime))
                // 当前时间戳
                const now = Date.parse(new Date())
                // 相差的毫秒数
                const msec = end - now
                if (msec < 0) {
                    activeFinish().then(() => {
                        this.$message({
                            message: '活动结束！',
                            type: 'warning'
                        });
                    })
                    this.isEnd = true;
                } else {
                    // 计算时分秒数
                    let day = parseInt(msec / 1000 / 60 / 60 / 24)
                    let hr = parseInt(msec / 1000 / 60 / 60 % 24)
                    let min = parseInt(msec / 1000 / 60 % 60)
                    let sec = parseInt(msec / 1000 % 60)
                    // 个位数前补零
                    hr = hr > 9 ? hr : '0' + hr
                    min = min > 9 ? min : '0' + min
                    sec = sec > 9 ? sec : '0' + sec
                    this.value = hr + ':' + min + ':' + sec
                    // 一秒后递归
                    setTimeout(this.countdown, 1000)
                }
            },
            show() {
                showGoods().then(request => {
                    console.log(request)
                    if (request.state === 1) {
                        this.ifexit = true;
                        let data = request.data;
                        this.endTime = data.endtime;
                        this.form.name = data.courseName;
                        this.url = data.image;
                        this.srcList[0] = data.image;
                        this.form.amount = data.amount;
                        this.form.teacherUser = data.username;
                        this.form.liveType = data.liveType;
                        this.percentage = (data.surplus / data.quantity).toFixed(2) * 100;
                        if (this.percentage === 0) {
                            this.isEnd = true;
                        }
                        this.countdown();
                    } else {
                        this.isEnd = true
                        this.$message('活动未创建！');
                    }
                }).catch(reason => {
                    this.$message('活动未创建！');
                })
            }


        }
    }
</script>

<style scoped>
  .line {
    text-align: center;
  }

  .volume {
    margin-top: 10px;
  }
</style>

