<template>
  <div v-loading="!proposal" class="poll-detail">
    <el-container>
      <el-main class="header">
        <div id="back-button" @click="$router.push({path: '/'})">
          <i class="el-icon-arrow-left"></i>View All Polls
        </div>
        <h1>{{proposal.proposal.title}}</h1>
        <p>
          <span>{{`${proposal.proposal.proposal_name} by ${proposal.proposal.proposer} `}}</span>
          <span style="margin: 0 5px">{{proposal.proposal.expires_at}} </span>
          <span>{{proposal.proposal.proposal_json.type || 'unknown'}}</span>
        </p>
        <div style="margin-bottom:30px">
          <div
            class="radio-button"
            @click="activeButton = 'desc'"
            :class="{'radio-button-active': activeButton === 'desc'}">
            Description
          </div>
          <div
            v-if="votes.length"
            class="radio-button"
            @click="turnTo('stats')"
            :class="{'radio-button-active': activeButton === 'stats'}">
            Stats
          </div>
          <div
            class="radio-button"
            @click="activeButton = 'voters'"
            :class="{'radio-button-active': activeButton === 'voters'}">
            Voters
          </div>
          <div
            class="radio-button"
            @click="turnTo('comments')"
            :class="{'radio-button-active': activeButton === 'comments'}">
            Comments
          </div>
        </div>
      <el-container>
        <el-main class="main" style="padding-top: 0;padding-left:0">
          <div v-if="activeButton !=='voters'" v-html="content" ref="desc" class="card prop-content">
          </div>
          <!-- <div class="card">
            <div class="radio-button" :class="{'radio-button-active': true}">English</div>
            <div class="radio-button" :class="{'radio-button-active': false}">中文</div>
          </div> -->
          <div v-else class="card" ref="voters">
            <h2>Voters</h2>
            <el-table :data="showVoters" :default-sort="{prop: 'staked', order: 'descending'}">
              <el-table-column sortable label="Name" prop="voter"></el-table-column>
              <el-table-column sortable label="Votes" prop="staked"></el-table-column>
              <el-table-column sortable label="Type" prop="type"></el-table-column>
              <el-table-column sortable label="Vote" prop="result"></el-table-column>
            </el-table>
            <div>
              <div class="button" style="margin: 20px auto;padding: 5px 20px" @click="showMoreVoters">Load more voters</div>
            </div>
          </div>
          <div v-if="votes.length" v-loading="chartLoading" class="card" ref="stats">
            <IEcharts
              style="width:800px;height:500px;margin:auto"
              :option="chartOption"
            ></IEcharts>
          </div>

          <div class="card" ref="comments">
            <h2>Auditor comments {{comments.length}}</h2>
            <Comment v-for="(comment, index) in comments" :key="index" v-bind="comment"></Comment>
          </div>

          <div class="card">
            <h2>BP comments {{comments.length}}</h2>
            <Comment v-for="(comment, index) in comments" :key="index" v-bind="comment"></Comment>
          </div>

          <div class="card">
            <h2>Other comments {{comments.length}}</h2>
            <Comment v-for="(comment, index) in comments" :key="index" v-bind="comment"></Comment>
          </div>
        </el-main>
        <el-aside width="350px">
          <div class="card" id="poll-status">
            <h2>Poll Status</h2>
            <el-progress :stroke-width="10" class="pass-percent" :percentage="agreePercent"></el-progress>
            <el-progress  :stroke-width="10" class="dissent-percent" :percentage="rejectPercent"></el-progress>
             <el-progress  :stroke-width="10" class="abstain-percent" :percentage="100-rejectPercent-agreePercent"></el-progress>
            <p>{{(this.proposal.stats.staked.total / 10000).toFixed(0)}} BOS voted</p>
            <div class="scatter-panel">
              <div v-if="scatter">
                <div v-if="isExpired(proposal.proposal.expires_at)">
                  This proposal is expired
                </div>
                <div v-else-if="!scatter.identity" @click="getIdentity" class="button">
                  Link Scatter to vote
                </div>
                <div v-else>
                  <div style="magin-bottom:10px">
                  <el-radio-group v-model="voteActionParams.vote">
                    <el-radio :label="1">YES</el-radio>
                    <el-radio :label="0">NO</el-radio>
                    <el-radio :label="255">ABSTAIN</el-radio>
                  </el-radio-group>
                  </div>
                  <div @click="sendVote" class="button" style="margin-right: 20px">
                    Vote
                  </div>
                  <div style="margin:5px 0">
                    <el-checkbox v-model="writeComment">
                      Post a public comment (optional)
                    </el-checkbox>
                    <div v-if="writeComment">
                      <el-input
                        type="textarea"
                        :rows="4"
                        v-model="myComment"
                      ></el-input>
                      <p>Select Yes/No to cast your vote and make your comment public on the EOS blockchain. Your comment and vote will be recorded on-chain for ever, if you want to change your comment please vote again and our algorithm will attempt to just show your latest comment.</p>
                    </div>
                  </div>
                  <div v-if="myVote">
                    <div class="button" @click="sendUnvote" style="background: red;margin-right: 20px">Unvote</div>
                    <span>You voted {{myVote.result}}</span>
                  </div>
                </div>
              </div>
              <a v-else target="blank" href="https://get-scatter.com/">
                <div class="button">
                  Get Scatter to vote
                </div>
              </a>
            </div>
            <hr style="border: none; border-bottom:1px solid #D8D8D8;" />
            <div>
              <p>{{this.proposal ? this.proposal.stats.votes.accounts : 0}} accounts</p>
              <p>{{this.proposal ? this.calcDays(this.proposal.proposal.created_at,new Date().toString()) : 0}} days since poll started</p>
              <p>{{this.proposal ? (this.proposal.stats.staked.total / 1e+11).toFixed(2) : 0}}% participation</p>
              <p>{{this.proposal ? this.yesLeadPercent : 0}}% YES lead over NO</p>
            </div>
          </div>
          <h2>
            Related Polls
          </h2>
          <div
            v-for="(prop, index) in relatedPolls"
            @click="turnDetail(prop)"
            :key="index"
            :style="{'margin-bottom': '30px', 'cursor': 'pointer'}"
          >
          <PropCard
            :type="prop.proposal.proposal_json.type || 'unknown'"
            :title="prop.proposal.title"
            :desc="prop.proposal.proposal_json.content || ''"
            :votes="prop.stats.votes"
            :staked="prop.stats.staked.total"
            ></PropCard>
          </div>
        </el-aside>
      </el-container>
      </el-main>
    </el-container>
  </div>
</template>

<script>
import marked from 'marked'
import Eos from 'eosjs'
import { MessageBox } from 'element-ui'
import { NETWORK } from '@/assets/constants.js'
import IEcharts from 'vue-echarts-v3/src/lite.js'
import 'echarts/lib/chart/pie'
import 'echarts/lib/component/tooltip'
import 'echarts/lib/component/title'
import PropCard from '@/components/PropCard.vue'
import Comment from '@/components/Comment.vue'
export default {
  name: 'PollDetail',
  components: {
    PropCard,
    IEcharts,
    Comment
  },
  computed: {
    chartLoading () {
      if (this.votes) {
        return false
      } else {
        return true
      }
    },
    chartOption () {
      let data = []
      if (this.votes) {
        this.votes.forEach(vote => {
          let label = true
          if (vote.vote !== 1 && vote.vote !== 0) {
            return
          }
          if (data.length > 20) {
            if (this.proposal.stats.staked.total !== 0 && (vote.staked * 10000 / this.proposal.stats.staked.total) > 0.002) {
              return
            }
            label = false
          }
          data.push({
            value: vote.staked,
            name: vote.voter,
            itemStyle: {
              color: (vote.vote === 1) ? 'rgb(97, 169, 19)' : 'rgb(217, 83, 79)'
            },
            label: {
              show: label
            },
            labelLine: {
              show: label
            }
          })
        })
      }
      return {
        title: {
          text: 'Votes by vote size',
          textStyle: {
            color: '#235894'
          },
          left: 'center'
        },
        tooltip: {
          trigger: 'item',
          formatter: '{b} : {c} BOS ({d}%)'
        },
        series: [
          {
            type: 'pie',
            radius: '65%',
            center: ['50%', '50%'],
            selectedMode: 'single',
            data: data,
            label: {
              normal: {
                textStyle: {
                  fontSize: 18,
                  color: '#235894'
                }
              }
            },
            itemStyle: {
              normal: {
                borderWidth: 1,
                borderColor: '#fff'
              }
            }
          }
        ]
      }
    },
    votes () { // votes for this proposal
      let allVotes = this.$store.state.votes || localStorage.getItem('votes')
      let allAccounts = this.$store.state.accounts || localStorage.getItem('accounts')
      let allProxies = this.$store.state.proxies || localStorage.getItem('proxies')
      if (allVotes && allAccounts && allProxies) {
        if (typeof allVotes === 'string') {
          allVotes = JSON.parse(allVotes)
        }
        let votes = []
        allVotes.forEach(vote => {
          if (vote.proposal_name === this.proposalName) {
            if (vote.vote === 1) {
              vote.result = 'YES'
            } else if (vote.vote === 0) {
              vote.result = 'NO'
            } else {
              vote.result = 'ABSTAIN'
            }
            if (allAccounts[vote.voter]) {
              vote.type = 'Voter'
              vote.staked = Number((allAccounts[vote.voter].staked / 10000).toFixed(0))
            } else if (allProxies[vote.voter]) {
              vote.type = 'Proxy'
              vote.staked = Number((allProxies[vote.voter].votes[this.proposalName].staked_proxy / 10000).toFixed(0))
            } else {
              vote.type = 'Voter'
              vote.staked = 0
            }
            if (votes.length === 0 || vote.staked <= votes[votes.length - 1].staked) {
              votes.push(vote)
            } else {
              for (let i = votes.length - 1; i >= 0; i--) {
                if (vote.staked < votes[i].staked) {
                  votes.splice(i + 1, 0, vote) // 插在该元素后面
                  break
                } else if (i === 0) {
                  votes.splice(0, 0, vote)
                }
              }
            }
          }
        })
        return votes
      } else {
        return []
      }
    },
    showVoters () {
      return this.votes.slice(0, this.showVotersNum)
    },
    myVote () {
      let myVotes = null
      if (this.$store.state.accounts && this.scatter && this.scatter.identity) {
        myVotes = this.$store.state.accounts[this.scatter.identity.accounts[0].name]
        if (myVotes && myVotes.votes[this.proposalName]) {
          let vote = { ...myVotes.votes[this.proposalName] }
          if (vote.vote === 1) {
            vote.result = 'YES'
          } else if (vote.vote === 0) {
            vote.result = 'NO'
          } else {
            vote.result = 'ABSTAIN'
          }
          return vote
        }
      }
      return null
    },
    comments () {
      let comments = []
      this.votes.forEach(vote => {
        if (vote.vote_json && vote.vote_json.comment) {
          comments.push({
            avatar: '',
            name: vote.voter,
            time: vote.updated_at,
            comment: vote.vote_json.comment
          })
        }
      })
      return comments
    },
    proposalName () {
      return this.$route.query.proposal || localStorage.getItem('proposalName')
    },
    proposal () {
      if (!this.$store.state.currentProposal) {
        this.$store.dispatch('getProposals', {
          proposalName: this.proposalName
        })
      }
      const defaultProp = {
        id: '',
        proposal: {
          expires_at: '',
          created_at: '',
          proposal_json: {
            type: '',
            content: ''
          },
          title: '',
          proposer: '',
          proposal_name: ''
        },
        stats: {
          votes: {},
          accounts: {},
          staked: {
            total: 0
          }
        }
      }
      return this.$store.state.currentProposal || defaultProp
    },
    relatedPolls () {
      let related = []
      const proposals = this.$store.state.proposals
      if (proposals && this.proposal) {
        Object.keys(proposals).forEach(key => {
          if (proposals[key].proposal.proposer === this.proposal.proposal.proposer &&
          proposals[key].proposal.proposal_name !== this.proposal.proposal.proposal_name) {
            if (related.length < 2) {
              related.push(proposals[key])
            }
          }
        })
      }
      return related
    },
    content () {
      if (this.proposal) {
        return marked(this.proposal.proposal.proposal_json.content, { sanitize: true })
      } else {
        return 'no content'
      }
    },
    agreePercent () {
      if (!this.proposal || this.proposal.stats.votes.total === 0 || !this.proposal.stats.votes[1]) {
        return 0
      } else {
        return Number((100 * this.proposal.stats.votes[1] / this.proposal.stats.votes.total).toFixed(1))
      }
    },
    rejectPercent () {
      if (!this.proposal || this.proposal.stats.votes.total === 0 || !this.proposal.stats.votes[0]) {
        return 0
      } else {
        return Number((100 * this.proposal.stats.votes[0] / this.proposal.stats.votes.total).toFixed(1))
      }
    },
    yesLeadPercent () {
      if (!this.proposal.stats.votes[0]) {
        return 100
      } else if (!this.proposal.stats.votes[1]) {
        return 0
      } else {
        return this.calcPercent(this.proposal.stats.votes[1] - this.proposal.stats.votes[0], this.proposal.stats.votes[1]).toFixed(0)
      }
    },
    scatter () {
      return this.$store.state.scatter
    },
    eos () {
      if (this.scatter && this.scatter.identity) {
        const eosOptions = { expireInSeconds: 60 }
        const eos = this.scatter.eos(NETWORK, Eos, eosOptions)
        return eos
      }
      return null
    }
  },
  data () {
    return {
      title: 'Should EOS tokens sent to eosio.ramfee and eosio.names accounts in the future be allocated to REX?',
      activeButton: 'desc',
      auditorsList: [],
      voteActionParams: {
        voter: '',
        proposal_name: '',
        vote: -1,
        vote_json: ''
      },
      myComment: '',
      writeComment: false,
      showVotersNum: 30

    }
  },
  mounted () {
  },
  methods: {
    getAuditors () {
      if (this.eos) {
        const tableOptions = {
          'scope': 'auditor.bos',
          'code': 'auditor.bos',
          'table': 'custodians',
          'json': true
        }
        this.eos.getTableRows(tableOptions).then(res => {
          this.auditorsList = res.rows
        })
      }
    },
    isExpired (exporiesAt) {
      let now = new Date().getTime() + (new Date().getTimezoneOffset() * 60 * 1000)
      let expiry = new Date(exporiesAt).getTime()
      if (expiry < now) {
        return true
      }
      return false
    },
    getIdentity () {
      const requiredFields = {
        accounts: [ NETWORK ]
      }
      this.scatter.getIdentity(requiredFields).then(() => {
        // console.log(this.scatter.identity)
        this.$store.dispatch('setScatter', { scatter: this.scatter })
      })
    },
    calcDays (start, end) {
      let startDay = new Date(start)
      let endDay = new Date(end)
      return ((endDay.getTime() - startDay.getTime()) / 1000 / 3600 / 24).toFixed(0)
    },
    calcPercent (numerator, denominator) {
      if (!denominator === 0) {
        return 0
      } else {
        return Number((numerator * 100 / denominator).toFixed(2))
      }
    },
    sendVote () {
      if (this.voteActionParams.vote === -1) {
        MessageBox.alert('Please choose your vote', '', {
          confirmButtonText: 'OK'
        })
      } else {
        const account = this.scatter.identity.accounts.find(x => x.blockchain === 'eos')

        this.voteActionParams.voter = account.name
        this.voteActionParams.proposal_name = this.proposalName
        if (this.myComment !== '' && this.writeComment) {
          this.voteActionParams.vote_json = JSON.stringify({ comment: this.myComment })
        }
        const transactionOptions = {
          actions: [{
            account: 'eosio.forum',
            name: 'vote',
            authorization: [{
              actor: account.name,
              permission: account.authority
            }],
            data: { ...this.voteActionParams }
          }]
        }
        this.eos.transaction(transactionOptions, { blocksBehind: 3, expireSeconds: 30 })
          .then(res => {
            MessageBox.alert(`Your vote has been cast on ${this.proposalName}`, '', {
              confirmButtonText: 'OK'
            })
          }).catch(e => {
            MessageBox.alert(JSON.parse(e).error.name, 'ERROR', {
              confirmButtonText: 'OK'
            })
          })
      }
      // eos.transfer(account.name, 'helloworld', '1.0000 EOS', 'memo', transactionOptions).then(trx => {
      //   // That's it!
      //   console.log(`Transaction ID: ${trx.transaction_id}`)
      // }).catch(error => {
      //   console.error(error)
      // })
    },
    sendUnvote () {
      const account = this.scatter.identity.accounts.find(x => x.blockchain === 'eos')
      const actionParams = {
        voter: account.name,
        proposal_name: this.proposalName
      }
      const transactionOptions = {
        actions: [{
          account: 'eosio.forum',
          name: 'unvote',
          authorization: [{
            actor: account.name,
            permission: account.authority
          }],
          data: { ...actionParams }
        }]
      }
      this.eos.transaction(transactionOptions, { blocksBehind: 3, expireSeconds: 30 })
        .then(res => {
          MessageBox.alert(`Your unvote on ${this.proposalName} was successful, data will be updated some time later`, '', {
            confirmButtonText: 'OK'
          })
        }).catch(e => {
          MessageBox.alert(JSON.parse(e).error.name, 'ERROR', {
            confirmButtonText: 'OK'
          })
        })
    },
    turnTo (target) {
      // 跳转到某个card
      this.activeButton = target
      this.$refs[target].scrollIntoView()
    },
    turnDetail (prop) {
      // 进入另一个prop
      if (window.localStorage) {
        localStorage.setItem('proposalName', prop.proposal.proposal_name)
      }
      this.$store.dispatch('setCurrentProposal', { proposal: prop })
      this.$router.push({ path: '/poll_detail', query: { proposal: prop.proposal.proposal_name } })
    },
    showMoreVoters () {
      this.showVotersNum += 30
      if (this.showVotersNum > this.votes.length) {
        this.showVotersNum = this.votes.length
      }
    }
  }
}
</script>

<style lang="stylus">
#poll-status
  .pass-percent
    .el-progress__text
      font-family Roboto-Bold
      font-size 11px
      color #30D094
      letter-spacing 0
      text-align center
    .el-progress-bar__inner
      background-image linear-gradient(270deg, #41B976 0%, #2CD69B 100%)
  .dissent-percent
    .el-progress__text
      font-family Roboto-Bold
      font-size 11px
      color #F46666
      letter-spacing 0
      text-align center
    .el-progress-bar__inner
      background-image linear-gradient(269deg, #F06262 0%, #FF7171 100%)
  .abstain-percent
    .el-progress__text
      font-family Roboto-Bold
      font-size 11px
      color #F4D03F
      letter-spacing 0
      text-align center
    .el-progress-bar__inner
      background-image linear-gradient(270deg, #F7DC6F 0%, #F1C40F 100%)

</style>

<style lang="stylus" scoped>
.poll-detail
  background-color rgb(232,236,255)
  padding 20px 0
  text-align left
  color: #507DFE;
.header
  h1
    font-family: Roboto-Medium;
    font-size: 30px;
    letter-spacing: 0;
  h2
    font-family: Roboto-Medium;
    font-size: 26px;
  p
    font-family: Roboto-Regular;
    font-size: 18px;
    color: #8A8A8A;
    letter-spacing: 0;
.main
  font-family: Roboto-Regular;
  font-size: 18px;
  color: #8A8A8A;
  letter-spacing: 0;
  h1, h2, h3, h4, h5
    font-family: Roboto-Medium;
    color: #507DFE;
.prop-content
  overflow-wrap break-word
  >>> h2
    color: #507DFE;
  >>> p
    font-family: Roboto-Regular;
    font-size: 18px;
    color: #8A8A8A;
    letter-spacing: 0;
#back-button
  cursor pointer
  width 160px
  height 32px
  line-height 32px
  background: #507DFE;
  border-radius: 15.5px;
  font-family: PingFangSC-Medium;
  font-size: 18px;
  color: #FFFFFF;
  letter-spacing: 0;
  text-align: center;
.card
  padding 22px 34px
  background: #FCFDFF;
  box-shadow: 0 2px 4px 0 #B0D9FF;
  border-radius: 8px;
  margin-bottom 22px
.radio-button
  height 24px
  min-width 103px
  line-height 24px
  margin-right 10px
  display inline-block
  cursor pointer
  opacity: 0.5;
  background: #7599FF;
  border-radius: 13.29px;
  font-family: PingFangSC-Medium;
  font-size: 13.71px;
  color: #FFFFFF;
  letter-spacing: 0;
  text-align: center;
.radio-button-active
  opacity 1
</style>
