<script>
	import { defineComponent, refs } from 'vue';
	import { IonHeader, IonContent, IonToolbar, IonTitle, IonMenuButton, IonPage, IonButtons, IonList, IonListHeader, IonLabel, IonItem, actionSheetController, IonNavLink, IonChip, IonSkeletonText, IonAvatar, IonRefresher, IonRefresherContent, IonCheckbox, IonButton, IonModal, IonDatetime, IonRippleEffect } from '@ionic/vue';

    import { checkmark, alertCircle } from 'ionicons/icons';

    import { Virtual } from 'swiper'
    import { Swiper, SwiperSlide } from 'swiper/vue';
    import 'swiper/css';

    import axios from 'axios';

    import HomeworkItemView from './HomeworkItemView.vue';

    import GetHomeworks from "@/functions/fetch/GetHomeworks.js";
    import GetToken from "@/functions/login/GetToken.js";

    import displayToast from '@/functions/utils/displayToast.js';
    import hapticsController from '@/functions/utils/hapticsController.js';

	export default defineComponent({
		name: 'FolderPage',
		components: {
			IonHeader,
			IonContent,
			IonToolbar,
			IonTitle,
			IonMenuButton,
			IonButtons,
            IonButton,
			IonList,
            IonPage,
            Swiper,
            SwiperSlide,
            IonRefresher,
            IonRefresherContent,
            IonCheckbox,
            IonModal,
            IonDatetime,
            IonItem,
            IonLabel,
            IonNavLink
		},
		setup() {
			return { 
                baseIndex: 250,
                minDate: require('@/functions/utils/datetimePicker.js').minCalendarDate(),
                maxDate: require('@/functions/utils/datetimePicker.js').maxCalendarDate(),
                isDateAvailable: require('@/functions/utils/datetimePicker.js').isDateAvailable,
				Virtual
			}
		},
		data() {
            const slides = Array.from({ length: this.baseIndex * 2 }).map(
                (el, index) => `Slide ${index}`
            );

			return {
                HomeworkItemView: HomeworkItemView,
				slides,
                currentIndex: this.baseIndex,
                rnButtonString: this.createDateString(this.$rn),
                loadedrnButtonString: this.createDateString(this.$rn),
                rnCalendarString: this.$rn.toISOString().split('T')[0],
                days: [],
                connected: false,
                shouldResetSwiper: false,
                rnPickerModalOpen: false,
                isChangingDate: false,
                isLoading: true,
			}
		},
		methods: {
            getHomeworks(force, goTo, event) {
                let startloading = setTimeout(() => {
                    this.isLoading = true;
                }, 500);

                for (let i = 0; i < 3; i++) {
                    let index = this.$refs.swiper.$el.swiper.realIndex + (i - 1);

                    // get index diff
                    let indexDiff = this.baseIndex - index;

                    // get rn
                    let selectedRN = new Date();

                    if(goTo) {
                        selectedRN = new Date(this.$rn);
                    }

                    selectedRN.setDate(selectedRN.getDate() - indexDiff);

                    // if i is 1
                    if(i == 1) {
                        this.$rn = selectedRN;
                        this.rnButtonString = this.createDateString(this.$rn);
                        this.rnCalendarString = this.$rn.toISOString().split('T')[0];
                    }

                    // get homeworks for rn
                    GetHomeworks(selectedRN, selectedRN, force).then((homeworks) => {
                        if(i == 2) {
                            clearTimeout(startloading);
                            this.isLoading = false;

                            if(event) {
                                event.detail.complete();
                            }
                        }

                        if (homeworks.error) {
                            if(homeworks.error == "ERR_BAD_REQUEST") {
                                if(this.days[index]) {
                                    this.days[index].loading = true;
                                }
                            }
                            else {
                                this.days[index] = [];
                                if(this.days[index]) {
                                    this.days[index].error = homeworks.error;
                                }
                            }
                        }
                        else {
                            this.days[index] = this.editHomeworks(homeworks);
                            this.loadedrnButtonString = this.createDateString(this.$rn);
                            if(this.days[index]) {
                                this.days[index].loading = false;
                            }
                        }
                    });
                }
            },
            editHomeworks(homeworks) {
                // for each homework
                for (let i = 0; i < homeworks.length; i++) {
                    // set homework to edit
                    let homework = homeworks[i];

                    // remove <br/> tags from homework.homework.content
                    homeworks[i].homework.shortContent = homework.homework.shortContent.replace(/[<]br[^>]*[>]/gi,"");
                }

                // set homeworks to edit
                return homeworks;
            },
            createDateString(date) {
                let dateObject = new Date(date);
                let day_string = dateObject.toLocaleString('default', { weekday: 'long' }).slice(0, 3);
                // return string like "jeu. 1"
                return day_string + ". " + dateObject.getDate();
            },
			rnInputChanged() {
                if(!this.isChangingDate) {
                    // get new date from rnInput
                    let newDate = new Date(this.$refs.rnInput.$el.value);

                    // update rn
                    this.$rn = newDate;

                    // reset swiper
                    this.$refs.swiper.$el.swiper.slideTo(this.baseIndex, 0, false);

                    // emit event
                    document.dispatchEvent(new CustomEvent('rnChanged', { detail: newDate }));
                }
                else {
                    this.isChangingDate = false;
                }
            },
            confirmRnInput() {
                this.changernPickerModalOpen(false);
            },
            openRnPicker() {
                this.$refs.rnPickerModal.$el.present();
            },
            resetSwiper() {
                // this.$refs.swiper.$el.swiper.slideTo(1, 0);
                return false;
            },
            handleRefresh(event) {
                this.getHomeworks(true, false, event);
            },
            changernPickerModalOpen(state) {
                this.rnPickerModalOpen = state;
            },
            changeDone(hw) {
                // microinteractions
                hapticsController.notification('success');

                // send request
                if(!this.dontRetryCheck) {
                    let homeworkID = hw.data.id;

                    const API = this.$api;
                    let token = localStorage.getItem('token');

                    let dayRequest = new Date(this.$rn);
                    let dayString = dayRequest.toISOString().split('T')[0];

                    let URL = `${API}/homework/changeState`;

                    // post request with token, homeworkId, dateFrom and dateTo
                    axios.post(URL, {
                        token: token,
                        homeworkId: homeworkID,
                        dateFrom: dayString,
                        dateTo: dayString
                    }).then(() => {
                        let checkboxID = `checkbox_${hw.data.id}`;
                        let checkbox = document.getElementById(checkboxID);

                        if(checkbox.checked) {
                            displayToast.presentToastFull(
                                "Devoir marqué comme fait",
                                `Votre devoir de ${hw.homework.subject} a été marqué comme fait.`,
                                "success",
                                checkmark
                            )
                        }
                        else {
                            displayToast.presentToastFull(
                                "Devoir marqué comme non fait",
                                `Votre devoir de ${hw.homework.subject} a été marqué comme non fait.`,
                                "success",
                                checkmark
                            )
                        }

                        // reset homework cache
                        localStorage.removeItem('HomeworkCache');

                        // reload homeworks
                        this.getHomeworks();
                    })
                    .catch((error) => {
                        let response = error.response.data;

                        // untick checkbox
                        let checkboxID = `checkbox_${hw.data.id}`;
                        let checkbox = document.getElementById(checkboxID);

                        setTimeout(() => {
                            this.dontRetryCheck = true;
                            checkbox.checked = !checkbox.checked;

                            setTimeout(() => {
                                this.dontRetryCheck = false;
                            }, 100);
                        }, 200);

                        if(response == "expired" || response == "notfound") {
                            displayToast.presentToastFull(
                                "Impossible de marquer ce devoir comme fait",
                                "Le token à expiré",
                                "danger",
                                alertCircle,
                            )

                            GetToken();
                        }
                        else if(response.status == "not found") {
                            displayToast.presentToastFull(
                                "Impossible de marquer ce devoir comme fait",
                                "Nous n'avons pas pu trouver ce devoir sur nos serveurs.",
                                "danger",
                                alertCircle,
                                true,
                                response.error
                            )
                        }
                        else if(response.status == "error") {
                            displayToast.presentToastFull(
                                "Impossible de marquer ce devoir comme fait",
                                "Une erreur est survenue lors de la requête.",
                                "danger",
                                alertCircle,
                                true,
                                response.error
                            )
                        }
                        else {
                            displayToast.presentToastFull(
                                "Impossible de marquer ce devoir comme fait",
                                "Erreur inconnue",
                                "danger",
                                alertCircle,
                                true,
                                error
                            )
                        }
                    });
                }
            }
		},
		mounted() {
            document.addEventListener('rnChanged', (e) => {
                this.rnButtonString = this.createDateString(e.detail);
                this.getHomeworks(false, e.detail);
            });

            this.getHomeworks();

            document.addEventListener('tokenUpdated', () => {
                this.getHomeworks();
                return false;
            });

            // detect swiper slide change
            let swiper = this.$refs.swiper.$el.swiper;

            swiper.on('slideChangeTransitionEnd', () => {
                // reset swiper
                this.resetSwiper()
                // isChangingDate
                this.isChangingDate = true;
                // emit event
                this.getHomeworks();
            });

            swiper.on('activeIndexChange', () => {
                this.currentIndex = swiper.activeIndex;
            });
		}
	});
</script>

<template>
	<ion-page ref="page">
		<IonHeader class="AppHeader" translucent>
			<IonToolbar>
				<ion-buttons slot="start"  mode="md">
					<ion-menu-button color="dark" mode="md"></ion-menu-button>
				</ion-buttons>

				<ion-title mode="md">Travail à faire</ion-title>

                <ion-buttons slot="end">
                    <ion-button mode="md" id="rnPickerModalButton" color="dark" @click="changernPickerModalOpen(true)">
                    <span class="material-symbols-outlined mdls" slot="start">calendar_month</span>

                    <p>{{ rnButtonString }}</p>
                    </ion-button>
                </ion-buttons>

                <ion-progress-bar type="indeterminate" v-if="isLoading"></ion-progress-bar>
			</IonToolbar>
		</IonHeader>

		<ion-content :fullscreen="true" class="content">

            <ion-refresher slot="fixed" @ionRefresh="handleRefresh($event)">
                <ion-refresher-content></ion-refresher-content>
            </ion-refresher>

            <div id="noTouchZone"></div>

            <swiper class="swiper" ref="swiper" :modules="[Virtual]" virtual :initialSlide="baseIndex" :speed="200" :spaceBetween="10" :preventClicks="true">
                <swiper-slide class="slide"
                v-for="(slideContent, index) in slides"
                :key="index"
                :virtualIndex="index">

                    <IonList>
                        <IonItem button v-for="homework in days[`${index}`]" :key="homework.id">
                            <div slot="start">
                                <ion-checkbox :id="`checkbox_${homework.data.id}`" :checked="homework.data.done" @ionChange="changeDone(homework)"></ion-checkbox>
                            </div>
                            
                            <IonNavLink class="navLink"  router-direction="forward" :component="HomeworkItemView" :componentProps="{urlHw: encodeURIComponent(JSON.stringify(homework))}">
                                <IonLabel :style="`--courseColor: ${homework.data.color};`" class="ion-text-wrap">
                                    <p><span class="courseColor"></span> {{ homework.homework.subject }}</p>
                                    <h5 class="hwContent">{{ homework.homework.shortContent }}</h5>
                                </IonLabel>
                            </IonNavLink>
                        </IonItem>
                    </IonList>

                    <div v-if="days[`${index}`]">
                        <div class="NoCours" v-if="days[`${index}`].length == 0 && !days[`${index}`].error && !days[`${index}`].loading">
                            <h1>😎</h1>
                            <h2>Aucun devoir pour ce jour</h2>
                            <p>Sélectionnez un autre jour dans le calendrier ou balayez l’écran pour changer de journée.</p>

                            <ion-button mode="md" fill="clear" @click="changernPickerModalOpen(true)" class="changeDayButton">Ouvrir le calendrier</ion-button>
                        </div>

                        <div class="NoCours" v-if="days[`${index}`].length == 0 && days[`${index}`].error == 'ERR_NETWORK' && !days[`${index}`].loading && !connected">
                            <h1>🌏</h1>
                            <h2>Pas de connexion à Internet</h2>
                            <p>Vous pouvez uniquement consulter les journées déjà chargées préalablement lorsque vous êtes hors-ligne.</p>
                        </div>

                        <div class="NoCours" v-if="days[`${index}`].length == 0 && days[`${index}`].error == 'ERR_NETWORK' && !days[`${index}`].loading && connected">
                            <h1>🛠️</h1>
                            <h2>Serveurs indisponibles</h2>
                            <p>Vous pouvez uniquement consulter les journées déjà chargées préalablement. Nos serveurs seront bientôt de nouveaux disponibles.</p>
                        </div>
                    </div>
                    <div v-else>
                        <div class="loadingSpin">
                            <IonSpinner></IonSpinner>
                        </div>
                    </div>

                </swiper-slide>
            </swiper>


            <IonModal :is-open="rnPickerModalOpen" ref="rnPickerModal" class="datetimeModal" @didDismiss="changernPickerModalOpen(false)" :keep-contents-mounted="true" :initial-breakpoint="0.55" :breakpoints="[0, 0.55]">
                <IonHeader>
                    <IonToolbar>
                    <ion-title>Sélection de la date</ion-title>
                    <ion-buttons slot="end">
                        <ion-button @click="confirmRnInput()">Terminé</ion-button>
                    </ion-buttons>
                    </IonToolbar>
                </IonHeader>
                <ion-content>
                    <IonDatetime 
                        presentation="date"
                        ref="rnInput"
                        size="cover"
                        :value="rnCalendarString"
                        :firstDayOfWeek="1"
                        :min="minDate"
                        :max="maxDate"
                        :is-date-enabled="isDateAvailable"
                        @ionChange="rnInputChanged()"
                    >
                    </IonDatetime>
                </ion-content>
            </IonModal>
		</ion-content>
	</ion-page>
</template>
  
<style scoped>
    .swiper {
        height: 100%;
        
        overflow: hidden;
        padding-top: 8px !important;

        margin-bottom: 0;
    }


	.loadingSpin {
        width: 100%;
        min-height: 40vh;
        height: 100%;

        display: flex;
        justify-content: center;
        align-items: center;
    }

    .courseColor {
        width: 10px;
        height: 10px;
        border-radius: 50%;
        background-color: var(--courseColor);
        display: inline-block;
        margin-right: 5px;
    }

    .changeDayButton {
        margin-top: 16px !important;
    }

    .hwContent {
        font-size: 16px;
    }

    #noTouchZone {
        width: 20px !important;
    }

    .navLink {
        margin: 10px 0px;
    }
</style>
