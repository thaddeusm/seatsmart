<template>
	<div id="sessionContainer">
        <transition name="fade">
            <aside id="leftPanel">
                <h1>
                	{{ session.activity.name }}
                </h1>
                <TitleBar v-if="session.chart !== 'anonymous' && loaded && chartExists" :classID="session.chart" :compact="true" :link="true" />
                <div id="illustrationArea" v-if="loaded">
                	<img v-if="session.activity.activityType == 'survey'" src="@/assets/survey-illustration.svg" alt="survey illustration">
                	<img v-else src="@/assets/response-pool-illustration.svg" alt="response pool illustration">
                </div>
                <h5>
                	{{ makePrettyDate(session.date._d) }}
                </h5>
                <h6 id="responses" v-if="session.responses.length > 1">
                	{{ session.responses.length }} responses
                </h6>
            </aside>
        </transition>
        <main>
            <header>
                <section id="backArea">
        			<button class="back-button" @click="routeBack"><img class="back-arrow" src="@/assets/backarrow.svg" alt="back arrow"> back</button>
        		</section>
            </header>
            <section v-if="loaded && session.activity.activityType == 'response pool' && !responsesMissing" class="response-pool">
                <ul v-if="session.responses.length > 0" id="responsePoolList">
                    <li v-for="(response, index) in session.responses" v-if="response.response.text" :key="`response${index}`">
                        <h4>{{ response.response.text }}</h4>
                        <h5 v-if="response.respondent !== 'anonymous'"><router-link :to="`/student/${response.respondent.id}`">
                            {{ response.respondent.firstName }} 
                            {{ response.respondent.lastName }}
                        </router-link></h5>
                    </li>
                </ul>
            </section>
            <section v-else-if="responsesMissing" class="response-pool">
                <img src="@/assets/database-bug.svg" alt="error icon" id="errorIcon">
                <div class="personal-note">
                    <img class="profile-image" src="@/assets/thaddeus.png" alt="sketched image of Thaddeus">
                    <div class="text-bubble">
                        <p>
                            Due to a bug at the beginning of the Fall 2019 semester, some activity records were not correctly stored in the database.  Otherwise, you might have ended an activity before responses were received.
                        </p>
                        <p>
                            If you think you have found a different problem, please <button class="email-link" @click="openEmail">contact me</button>.
                        </p>
                    </div>
                </div>
            </section>
            <section v-else-if="loaded && session.activity.activityType == 'survey'">
                <div class="survey" v-if="session.responses.length > 0">
                    <vc-donut
                        background="black" foreground="black"
                        :size="250" unit="px" :thickness="20"
                        has-legend legend-placement="bottom"
                        :sections="donutSections" :total="session.responses.length"
                    >
                    </vc-donut>
                </div>
                <div id="responseArea">
                    <h2>
                        Individual Responses
                    </h2>
                    <sequential-entrance fromTop delay="20">
                        <div 
                            v-for="(response, index) in session.responses" 
                            :key="`response${index}`"
                            class="response-card"
                        >
                            <section 
                                class="response-header"
                            >
                                <div 
                                    class="circle" 
                                    :style="{background: findSpectrumColor(response.response.choice)}">
                                </div>
                            </section>
                            <section class="response-body">
                                <h5><router-link :to="`/student/${response.respondent.id}`">
                                    {{ response.respondent.firstName }} 
                                    {{ response.respondent.lastName }}
                                </router-link></h5>
                                <p>{{ response.response.choice }}</p>
                            </section>
                        </div>
                    </sequential-entrance>
                </div>
            </section>
        </main>
        <!-- <TouchBar :show="!modalOpen && !alertModalOpen" :bar="[
            {type: 'spacer', size: 'flexible'},
            {type: 'button', label: '⭐️', method: function() {toggleSelected()}},
            {type: 'spacer', size: 'flexible'},
			{type: 'button', label: 'Toggle Trends', method: function() {toggleTrendScope()}},
            {type: 'spacer', size: 'flexible'},
			{type: 'button', label: '📝', method: function() {openModal('note')}}
	       ]" 
           :escapeItem="{type: 'button', label: 'Back', method: function() {routeBack()}}"
        /> -->
    </div>
</template>

<script>
const shell = require('electron').shell

import db from '@/db.js'
import moment from 'moment'

moment.updateLocale("en", { week: {
  dow: 1, // First day of week is Monday
  doy: 6  // First week of year must contain 1 January (7 + 0 - 1)
}})

import TitleBar from '@/components/TitleBar.vue'
import TouchBar from '@/components/TouchBar.vue'

export default {
	name: 'SessionExplorer',
	props: {
		id: String
	},
	components: {
		TitleBar,
		TouchBar
	},
	data() {
		return {
			session: {
                date: {},
                activity: {},
                responses: [{
                    respondent: '',
                    response: ''
                }],
                chart: ''
            },
     		loaded: false,
            donutSectionColorSpectrum: [
                '#FCBB04',
                '#E5E5E5',
                '#D2360A',
                '#6C6C6C',
                '#B28402',
                '#FFFFFF',
                '#832206',
                '#FDCD48',
                '#F66239'
            ]
		}
	},
	computed: {
		chartExists() {
            let chart = this.allCharts[this.session.chart]

            if (chart !== undefined) {
                return true
            } else {
                return false
            }
        },
        allCharts() {
            return this.$store.state.allClasses
        },
        responsesMissing() {
            if (this.session.activity.activityType == 'response pool') {
                let missing = true

                for (let i=0; i<this.session.responses.length; i++) {
                    if (this.session.responses[i].response.text !== undefined) {
                        missing = false
                        break
                    }
                }

                return missing
            }

            return false
        },
        donutSections() {
            let responses = this.session.responses

            let sections = []

            // format survey choices into sections per donut chart API
            if (this.session.activity.activityType == 'survey') {
                for (let i=0; i<this.session.activity.content.choices.length; i++) {
                    let obj = {
                        value: 0,
                        label: this.session.activity.content.choices[i],
                        color: this.donutSectionColorSpectrum[i]
                    }

                    sections.push(obj)
                }

                // increment section values based upon participant responses
                for (let j=0; j<responses.length; j++) {
                    for (let k=0; k<sections.length; k++) {
                        if (sections[k].label == responses[j].response.choice) {
                            sections[k].value++
                            break
                        }
                    }
                }
            }

            return sections
        }
	},
	methods: {
		getSessionInfo() {
			db.readSomething('activitySessions', {_id: this.id})
                .then((results) => {
                    this.session = results[0]
                    this.loaded = true

                    console.log(this.session)
                })
		},
		routeBack() {
            this.$router.go(-1)
        },
        makePrettyDate(dateObj) {
            return moment(dateObj).format('dddd, MMM D')
        },
        findSpectrumColor(choice) {
            let index = this.session.activity.content.choices.indexOf(choice)

            return this.donutSectionColorSpectrum[index]
        },
        openEmail() {
            shell.openExternal('mailto:tbmccleary@fhsu.edu?subject=Seatsmart')
        }
	},
	mounted() {
		this.getSessionInfo()
	}
}
</script>

<style scoped>
@keyframes spin {
    from {transform: rotate(0);}
    to {transform: rotate(360deg);}
}

#sessionContainer {
    background: var(--black);
    width: 100%;
    height: 100%;
    display: grid;
    grid-template-columns: 25% 1fr;
    grid-template-rows: auto;
    grid-template-areas: "leftPanel main";
}

#backArea {
    margin-top: 7%;
    margin-bottom: 15px;
}

.back-button {
	font-family: "ArchivoNarrow";
	color: var(--black);
	font-size: 14px;
    margin-left: 5px;
}

.back-button > img {
	vertical-align: middle;
	padding-bottom: 2px;
}

#leftPanel {
    grid-area: leftPanel;
    color: var(--white);
    background: var(--black);
    text-align: center;
    top: 90px;
    height: 100%;
    position: fixed;
    width: 25%;
}

#illustrationArea {
	background: var(--light-gray);
	margin: 20px 0;
}

#responses {
	margin: 10px 0;
    color: var(--yellow);
}

main {
    grid-area: main;
    background: var(--light-gray);
    padding: 0 10%;
    min-height: 100vh;
    height: 100%;
}

#responseArea {
    margin: 50px auto 50px auto;
    text-align: center;
}

#responseArea > h2 {
    margin-bottom: 50px;
}

.response-pool {
    margin-top: 50px;
}

.response-pool > h3 {
    text-align: center;
    margin-bottom: 30px;
}

.survey {
    background: var(--black);
    color: var(--white);
    padding: 40px 0;
    border-radius: 5px;
}

button {
    background: none;
    outline: none;
    border: none;
    cursor: pointer;
}

ul {
    list-style: none;
}

#responsePoolList > li > h4 {
    background: var(--gray);
    color: var(--white);
    padding: 10px;
    margin: 50px 0 5px 0;
}

#responsePoolList > li > h5 {
    text-align: right;
    margin-bottom: 50px;
}

#individualizedResponses {
    text-align: center;
}

#individualizedResponses > li {
    display: inline-block;
    width: 200px;
    height: 150px;
    display: inline-grid;
    align-items: center;
    justify-content: center;
}

.hide-button {
    background: none;
    cursor: pointer;
    outline: none;
    border: none;
    display: block;
}

.hide-button > img {
    height: 20px;
    vertical-align: middle;
}

.show-all-button {
    display: block;
    font-size: 16px;
    padding: 5px 12px;
    border-radius: 5px;
    cursor: pointer;
    box-shadow: 0px 0px 1px var(--black);
    outline: none;
    background: var(--yellow);
    color: var(--black);
    margin: 10px auto;
}

.response-card {
    display: inline-block;
    width: 320px;
    margin: 40px 5%;
    padding-bottom: 10px;
    border-radius: 10px;
    box-shadow: 1px 2px 1px 2px var(--gray);
}

.response-header {
    margin-top: -25px;
    text-align: center;
}

.circle {
    display: inline-grid;
    justify-content: center;
    align-items: center;
    width: 35px;
    height: 35px;
    font-size: 22px;
    line-height: 22px;
    border-radius: 35px;
    margin: 0 3%;
    box-shadow: 0px 0px 5px var(--gray);
}

.response-body {
    padding: 10px;
}

.response-body > h5 {
    text-align: center;
    margin-bottom: 3px;
}

.response-body > p {
    font-weight: 100;
    background: var(--white);
    padding: 5px 12px;
    border-radius: 3px;
    margin: 15px 0 10px 0;
    text-align: left;
}

#errorIcon {
    display: block;
    width: 5em;
    margin: 50px auto;
}

p {
    margin: 10px 0;
}

.personal-note {
    display: grid;
    grid-template-rows: auto 1fr;
    grid-template-areas: "profile textBubble";
    margin: 30px 0;
}

.profile-image {
    grid-area: profile;
    width: 100px;
    margin-right: 10px;
    border-radius: 100px;
    background: var(--white);
    box-shadow: 0px 0px 5px var(--black);
}

.text-bubble {
    grid-area: textBubble;
    padding: 5px 15px;
    background: var(--dark-gray);
    color: var(--light-gray);
    border-radius: 5px;
}

.email-link {
    color: var(--yellow);
    font-size: 18px;
}

a {
    text-decoration: none;
    color: var(--black);
}

.fade-enter-active, .fade-leave-active {
	transition: opacity .2s;
}

.fade-enter, .fade-leave-to /* .fade-leave-active below version 2.1.8 */ {
	opacity: 0;
}
</style>