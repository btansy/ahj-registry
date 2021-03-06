<template>
    <div class="user-page">
        <div class="user-page-container" v-if="userInfoLoaded">
                <div class='user__header'>
                    <img class='banner__image' src="../assets/images/background-image.jpeg"/>
                        <img class='user__image' src="../assets/images/profile-image-default.jpeg" /> 
                    <div class="header__content">
                        <div class="header__content__left">
                            <div class="multi__info__line">
                                <template v-if="fullName !== ' '">
                                    <h2><b>{{fullName}}</b></h2>
                                </template>
                                <template v-else>
                                    <h2><b>{{username}}</b></h2>
                                </template>
                            </div>
                            <hr>
                            <h4 v-if="companyAffiliation !== null">{{companyAffiliation}}</h4>
                            <h6>Number of submitted edits: {{numSubmittedEdits}}</h6>
                            <h6>Number of accepted edits: {{numAcceptedEdits}}</h6>
                            <h6>Community Score: {{communityScore}}</h6>
                        </div>
                        <div id="header-content-right">
                            <div id="header-buttons-right">
                                <b-button v-b-modal.user-modal size="sm" class="contact-info-button" variant="outline-primary" busy="True">Contact Info</b-button>
                                <b-modal id="user-modal">
                                    <template #modal-title>
                                        {{`${fullName}'s Contact Info`}}
                                    </template>
                                    {{`Email: ${email}`}} <br>
                                    {{`Phone: ${phone}`}} <br>
                                    {{`Preferred Contact Method: ${preferredContact}`}}
                                </b-modal>
                                <b-button v-if="this.$store.getters.loggedIn && username === this.$store.state.loginStatus.Username" class="button" @click="$router.push({ name: 'settings'})" pill variant="primary">
                                    Edit Profile
                                </b-button>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="user__about">
                    <h2>About</h2>
                    <hr>
                    <template v-if="personalBio !== null && personalBio !== ''">
                        <p>{{personalBio}}</p>
                    </template>
                    <template v-else>
                        <p class="alternative-text">User has not filled their about section.</p>
                    </template>
                </div>
                <div class="user__feed">
                    <h2>Feed</h2>
                    <hr>
                    <b-button-group>
                        <b-button variant="outline-secondary" @click="GetUserActivity('Edit')"> Edits </b-button>
                        <b-button variant="outline-secondary" @click="GetUserActivity('Comment')"> Comments </b-button>
                    </b-button-group>
                    <template v-if="activities.length > 0">
                        <template v-if="FeedActivity === 'Edit'">
                            <activity-entry 
                                v-for="activity in (activities || []).slice(0, 10)" 
                                v-bind:key="activity.EditID" 
                                v-bind:UserData="activity.ChangedBy" 
                                v-bind:ActivityType="FeedActivity" 
                                v-bind:ActivityData="activity">
                            </activity-entry>
                        </template>
                        <template v-else>
                            <activity-entry 
                                v-for="activity in (activities || []).slice(0, 10)" 
                                v-bind:key="activity.CommentID.Value" 
                                v-bind:UserData="ProfileDataObj" 
                                v-bind:ActivityType="FeedActivity" 
                                v-bind:ActivityData="activity">
                            </activity-entry>
                        </template>
                    </template>
                    <template v-else>
                        <div v-if="this.gettingUserActivity">
                            <p class="alternative-text">Getting {{FeedActivity}} history...</p>
                        </div>
                        <div v-else>
                            <p class="alternative-text">User has no {{FeedActivity}} history</p>
                        </div>
                    </template>
                </div>
        </div>
    </div>
</template>

<script>
import axios from "axios";
import constants from "../constants.js";
import ActivityEntry from "../components/UserAccounts/UserProfile/ActivityEntry.vue";

export default {
    computed: {
        userID(){
            return this.ProfileData['UserID']; 
        },
        photo(){
            return this.ProfileData['Photo']; 
        },
        fullName(){
            return this.ProfileData['ContactID']['FirstName']['Value'] + " " + this.ProfileData['ContactID']['LastName']['Value']; 
        },
        email(){
            return this.ProfileData['Email'];
        },
        username(){
            return this.ProfileData['Username'];
        },
        phone(){
            return this.ProfileData['ContactID']['MobilePhone']['Value'];
        },
        preferredContact(){
            return this.ProfileData['ContactID']['PreferredContactMethod']['Value'];
        },
        url(){
            return this.ProfileData['ContactID']['URL']['Value'];
        },
        numReviewsDone(){
            return this.ProfileData['NumReviewsDone'];
        },
        numAcceptedEdits(){
            return this.ProfileData['AcceptedEdits'];
        },
        numSubmittedEdits(){
            return this.ProfileData['SubmittedEdits'];
        },
        communityScore(){
            return this.ProfileData['CommunityScore'];
        },
        personalBio(){
            return this.ProfileData['PersonalBio'];
        },
        companyAffiliation(){
            return this.ProfileData['CompanyAffiliation'];
        },
        isPeerReviewer(){
            return this.ProfileData['IsPeerReviewer'];
        },
        location(){
            return this.ProfileData['IsPeerReviewer'];
        },
        ProfileDataObj(){
            return this.ProfileData;
        },
        canSubmitPermit(){
            return this.$store.getters.loggedIn && !this.$store.state.loginStatus.MaintainedAHJs.includes(this.$route.params.AHJID);
        },
    },
    data() {
        return {
            ProfileData: {},
            userInfoLoaded: false,
            activities: [],
            FeedActivity: 'Edit',
            gettingUserActivity: true,
            uploadedApplication: null,
            applicationStatus: null,
        }
    },
    methods: {
        async GetUserInfo(){
            let query = constants.API_ENDPOINT + "user-one/" + this.$route.params.username;
            return axios.get(query, {
                    headers: {
                        Authorization: `${this.$store.getters.authToken}`
                    }
                })
                .then(response => {
                    this.ProfileData = response.data;
                    this.userInfoLoaded = true;
                    if (this.$store.getters.LoggedIn && this.$route.params.username === this.$store.state.LoginStatus.Username){
                        this.$store.commit("changeCurrentUserInfo", response.data);
                    }
                })
                .catch(() => {
                });
        },
        GetUserActivity(activityType) {
            this.gettingUserActivity = true;
            this.activities = [];
            this.FeedActivity = activityType;
            let query = activityType === 'Edit' ? constants.API_ENDPOINT + "user/edits/" : constants.API_ENDPOINT + "user/comments/"
            axios.get(query, {
                  params: {
                    'UserID': this.ProfileData.UserID
                  },
                  headers: {
                    Authorization: `${this.$store.getters.authToken}`
                  }
                })
                .then(response => {
                    this.gettingUserActivity = false;
                    this.activities = response.data;
                })
                .catch(() => {
                });
        },
        sendToMessaging(){
        },
    },
    mounted: async function() {
        await this.GetUserInfo();
        this.GetUserActivity('Edit');
    },
    components: {
    "activity-entry": ActivityEntry
  }

}

</script>

<style scoped>
.user-page {
    background-color: #f3f2ef;
}

hr {
    margin-top: 0px;
    margin-bottom: 5px;
}
li {
    list-style-type: none;
}
.user-page-container {
    width: 70%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    margin: 0 auto;
    margin-top: 40px;
}

.user-page-container > * {
    margin-top: 20px;
}

.user-page-container > *:first-child {
    margin-top: 0px;
}

.user__header {
    align-items: center;
    border: 1px solid lightgray;
    border-radius: 10px;
    background-color: white;
    padding-bottom: 10px;
}

.user__about {
    align-items: center;
    border: 1px solid lightgray;
    border-radius: 10px;
    background-color: white;
    padding: 20px;
}


.user__feed {
    align-items: center;
    border: 1px solid lightgray;
    border-radius: 10px;
    background-color: white;
    padding: 20px;
}

.user__about p {
    font-size: 18px;
}

.banner__image {
    margin-bottom: -80px;
    width: 100%;
    height: 200px;
    border-top-left-radius: 10px;
    border-top-right-radius: 10px;
    object-fit: cover;
}

.multi__info__line {
    display: flex;
    justify-content: flex-start;
    flex-wrap: wrap;
    align-items: center;
    min-width: 300px;
}

.multi__info__line > * {
    margin-left: 15px;
}

.multi__info__line > *:first-child {
    margin-left: 0px;
}

.header__content {
    display: flex;
    justify-content: space-between;
}

.header__content__left {
    flex: 0.4;
    margin-left: 250px;
}
.header__content__center {
    padding-top: 115px;
    flex: 0.3;
}
.community-score-text {
    margin-top: 20px;
}
.header__content__right {
    flex: 0.3;
}

#header-buttons-right {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    margin-right: 3em;
}
#header-buttons-right > button:first-child {
    margin-right: 1em;
    margin-bottom: 0;
}

.username-text {
    font-weight: 10;
}

.user__image {
    border-radius: 160px;
    border: 1.5px solid lightgray;
    height: 160px;
    width: 160px;
    object-fit: cover;
    margin-left: 50px;
    margin-bottom: -60px;
}

.header__buttons__right {
    display: flex;
    align-items: center;
    justify-content: flex-end;
}

.button {
    min-width: 130px;
    border: none;
    background-color: #ff8c00 !important;
}
.btn-outline-primary {
    color: #ff8c00 !important;
    border-color: #ff8c00 !important;
}
.btn-outline-primary:hover {
    color: white !important;
    background-color: #ff8c00 !important;
}

.check-icon {
    color: green;
}

.x-icon {
    color: red;
}

.more-icon {
    margin-right: 20px;
    cursor: pointer;
}

.check-icon-text {
}

.circle-icon {
    height: 8px;
    width: 8px;
    margin-bottom: 8px;
}

.contact-info-button {
    margin-bottom: 6px;
}

.alternative-text {
    color: gray;
}

.badgeIcon {
    border-radius: 60px;
    height: 60px;
    width: 60px;
    margin-left: 10px;
    margin-bottom: 5px;
}
.badgeIconRibbon {
    border-radius: 80px;
    height: 60px;
    width: 60px;
    object-fit: contain;
    margin-left: 10px;
    margin-bottom: 5px;
}

.user__ranking {
    align-items: center;
    border: 1px solid lightgray;
    border-radius: 10px;
    background-color: white;
    padding: 20px;
}

.user__ranking__container {
    display: flex;
    flex-direction: column;
    background-color: white;
    border: 1px solid lightgray;
    border-radius: 10px;
    padding: 20px;
}
.user__ranking__boards {
    display:flex;
    justify-content: space-evenly;
}
.user__ranking__boards > * {
    flex: 2;
    margin-top: 10px;
}
.ranking__board__container {
    display: flex;
    flex-direction: column;
    border-right: 0.5px solid grey;
    border-left: 0.5px solid grey;
}
.ranking__board {
    display: flex;
    flex-direction: column;
}
.ranking__board__container {
    text-align: center;
}
.ranking__board__container > hr {
    width: 60%;
    margin: auto;
    padding-bottom: 10px;
    margin-bottom: 0px;
}
.ranking__board__profileuser{
    display: flex;
    margin-left: 30px;
    align-items: center;
    padding-top: 15px;
    padding-bottom: 15px;
}
.ranking__board__profileuser > h5{
    font-size: 1.5em;
    font-weight: bold;
}
.ranking__board__profileuser > * {
    margin-right: 10px;
}
.ranking__board__user {
    display: flex;
    margin-left: 30px;
    align-items: center;
    padding-top: 5px;
    padding-bottom: 5px;
}
.ranking__board__user > * {
    margin-right: 10px;
}
.ranking__board__score {
    align-self: flex-end;
    flex: 1;
}
.ranking__board__user > img {
    border-radius: 40px;
    height: 40px;
    width: 40px;
    display: cover;
}
.ranking__board__profileuser > img {
    border-radius: 50px;
    height: 50px;
    width: 50px;
    display: cover;
}
.vertical-dots-icon-top {
    align-self: center;
    margin-bottom: 10px;
}
.vertical-dots-icon-bottom {
    align-self: center;
}
.ranking__board__username {
    flex: 3;
    text-align: left;
    margin-left: 20px;
}
.ranking__board__profileuser__username {
    flex: 3;
    text-align: left;
    margin-left: 10px;
}
.ranking__board__rank {
    flex: 1;
}

.ranking__board__stats{
    display: flex;
    flex-direction: column;
    margin-bottom: 10px;
}
.ranking__board__stats > *{
    margin: 0;
}
.ranking__board__stats > h5 {
    color: grey;
}

hr {
  margin-left: 0;
  margin-right: 0;
}

</style>