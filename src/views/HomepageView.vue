<script>
	import { defineComponent } from 'vue';
	import {
		IonHeader,
		IonContent,
		IonToolbar,
		IonTitle,
		IonMenuButton,
		IonPage,
		IonList,
		IonItem,
		IonLabel,
		IonListHeader,
		IonButton,
		IonSpinner,
		IonRefresher,
		IonChip,
		IonItemGroup,
		IonButtons,
		IonRefresherContent,
		IonSkeletonText,
		alertController,
		IonNavLink
	} from '@ionic/vue';

	import { NotificationBadge } from 'capacitor-notification-badge';

	import { StatusBar, Style } from '@capacitor/status-bar';

	import { Network } from '@capacitor/network';

	import UserView from './settings/UserView.vue';
	import InfoView from './news/InfoView.vue';
	import MarkView from './grades/MarkView.vue';
	import HomeworkItemView from './homework/HomeworkItemView.vue';
	
	import timetableEdit from '@/functions/utils/timetableEdit.js';
	import subjectColor from '@/functions/utils/subjectColor.js';

	// recap
	import GetRecap from "@/functions/fetch/GetRecap.js";

	export default defineComponent({
		name: 'FolderPage',
		components: {
			IonHeader,
			IonContent,
			IonToolbar,
			IonTitle,
			IonMenuButton,
			IonButtons,
			IonPage,
			IonList,
			IonListHeader,
			IonButton,
			IonItem,
			IonLabel,
			IonSpinner,
			IonRefresher,
			IonChip,
			IonItemGroup,
			IonRefresherContent,
			IonSkeletonText,
			IonNavLink
		},
		data() {
			return {
				connected: false,
				fakeTime: 200,
				timetable: [],
				ttbLoading: false,
				nextCoursTime: "",
				nextCoursStarted: false,
				nextCoursCompletion: 0,
				percentage: 0,
				updateTime: null,
				firstName: '',
				homeworks: [],
				hwLoading: true,
				blockChangeDone: false,
				editMode: false,
				noCoursesEmoji: this.randomEmoji(),
				noCoursesMsg: this.randomMsg(),
				userData: [],
				userName: "",
				avatar: "",
				grades: [],
				gradesLoading: true,
				UserView: UserView,
				news: [],
				newsLoading: true,
				internetConnection: true,
				allLoaded: false,
				showLoading: true,
				toolbarColor: "",
				InfoView: InfoView,
				MarkView: MarkView,
				HomeworkItemView: HomeworkItemView,
			}
		},
		methods: {
			goto(url) {
				document.dispatchEvent(new CustomEvent('navTransitionEnable'));
				this.$router.push(url);
			},
			randomEmoji() {
				let list = ["😊", "😎", "😴", "👌", "🌞", "📚", "💪", "💤", "😉", "🥱"]
				return list[Math.floor(Math.random() * list.length)];
			},
			randomMsg() {
				let list = [
					"Temps calme",
					"Pas de cours, on révise ?",
					"C'est la sieste (ou pas)",
					"Je suis sûr qu'il te reste des devoirs...",
					"Il n'y a jamais vraiment rien à faire !",
					"Il est temps de commencer ce joli DM !",
					"Il fait beau dehors ?",
					"Ça tombe bien, ce livre ne se finira pas tout seul !",
					"Flûte, le cours de maths est fini",
					"Après l'effort, le réconfort ;)",
					"Alors, ça se la coule douce ?",
					"Prenons de l'avance sur la semaine prochaine !",
					"Il est temps de reprendre la lecture !"
				]
				return list[Math.floor(Math.random() * list.length)];
			},
			editTimetable(timetable) {
				let now = new Date();
				let lessons = []

				timetable = timetableEdit(timetable);

				// add custom courses
				let customCourses = JSON.parse(localStorage.getItem('customCourses')) || [];
				customCourses.forEach((customCourse) => {
					// if course is in the same day
					let customDay = new Date(customCourse.day);
					let currentDay = new Date(this.$rn);

					let st = new Date(customCourse.course.time.start);
					let en = new Date(customCourse.course.time.end);

					// make st and en the same day as currentDay
					st.setDate(currentDay.getDate());
					st.setMonth(currentDay.getMonth());
					st.setFullYear(currentDay.getFullYear());

					en.setDate(currentDay.getDate());
					en.setMonth(currentDay.getMonth());
					en.setFullYear(currentDay.getFullYear());

					if (customDay.getDate() == currentDay.getDate() && customDay.getMonth() == currentDay
						.getMonth() && customDay.getFullYear() == currentDay.getFullYear()) {
						customCourse.course.time.start = st;
						customCourse.course.time.end = en;
						customCourse.course.course.color = subjectColor.getSubjectColor(customCourse.course
							.data.subject, subjectColor.getRandomColor());
						timetable.push(customCourse.course);
					}
				});
				
				// if 2 courses at the same time
				// remove all courses with sameTime = true
				timetable = timetable.filter((lesson) => {
					if (lesson.course.sameTime) {
						return false;
					} else {
						return true;
					}
				});

				// get next lesson (cours.time.start)
				lessons = timetable.filter((lesson) => {
					let lessonStart = new Date(lesson.time.start);
					let lessonEnd = new Date(lesson.time.end);

					// get minutes before next lesson
					let mins = Math.floor((lessonStart - now) / 1000 / 60);
					let gap = -((Math.floor((lessonEnd - lessonStart) / 1000 / 60)) / 2);

					if (lessons.length != 0) {
						return false;
					}

					// if less than 60 mins
					if (mins < 60 && mins >= 0) {
						if (mins > 1) {
							this.nextCoursTime = `dans ${mins} minutes`
						} else if (mins == 0) {
							this.nextCoursTime = `dans moins d'une minute`
						} else {
							this.nextCoursTime = `dans ${mins} minute`
						}

						this.nextCoursStarted = false;
					} else if (mins < 0) {
						if (lessonEnd <= now) {
							return false;
						}

						this.nextCoursStarted = true;

						// get percentage of lesson done
						let lessonTime = lessonEnd - lessonStart;
						let lessonTimeDone = now - lessonStart;
						let percentage = Math.floor((lessonTimeDone / lessonTime) * 100);

						this.nextCoursCompletion = percentage;

						// get minutes before lesson ends
						let endMins = Math.floor((lessonEnd - now) / 1000 / 60);

						this.nextCoursTime = `${endMins} min rest.`;
					} else {
						let hours = Math.floor(mins / 60);
						mins = mins % 60;

						if (mins > 1) {
							mins = `${mins} minutes`;
						} else {
							mins = `${mins} minute`;
						}

						if (hours > 1) {
							hours = `${hours} heures`;
						} else {
							hours = `${hours} heure`;
						}

						this.nextCoursTime = `dans ${hours} et ${mins}`;

						this.nextCoursStarted = false;
					}

					if (mins < gap) {
						return false;
					}

					lesson.hasStatus = lesson.status.status != undefined;

					lessons.push(lesson)
					return true;
				});

				// if lessons is empty but not timetable, get last lesson
				if (lessons.length == 0 && timetable.length > 0) {
					for (let i = timetable.length - 1; i >= 0; i--) {
						let lesson = timetable[i];

						let lessonEnd = new Date(lesson.time.end);
						let lessonStart = new Date(lesson.time.start);
						let endMins = Math.floor((lessonEnd - now) / 1000 / 60);
						let startMins = Math.floor((lessonStart - now) / 1000 / 60);

						let lessonDuration = -(Math.floor((lessonEnd - lessonStart) / 1000 / 60));

						if (endMins < -120) {
							continue;
						}

						if (startMins > lessonDuration) {
							this.nextCoursStarted = true;

							// get percentage of lesson done
							let lessonTime = lessonEnd - lessonStart;
							let lessonTimeDone = now - lessonStart;
							let percentage = Math.floor((lessonTimeDone / lessonTime) * 100);

							this.nextCoursCompletion = percentage;

							this.nextCoursTime = `${endMins} min rest.`;

							
							lesson.hasStatus = lesson.status.status != undefined;

							lessons.push(lesson);
							break;
						}

						this.nextCoursTime = "Cours terminé";
						this.nextCoursStarted = false;
						lesson.hasStatus = lesson.status.status != undefined;
						lessons.push(lesson);
						break;
					}
				}

				return lessons;
			},
			getRecap(force) {
				this.hwLoading = true;
				this.gradesLoading = true;
				this.newsLoading = true;
				this.allLoaded = false;

				GetRecap(force).then((recap) => {
					// loaded
					setTimeout(() => {
						this.allLoaded = true;
						this.showLoading = false;
					}, this.fakeTime);

					this.fakeTime = 0;

					// timetable
					const timetable = recap.timetable;
					this.timetable = this.editTimetable(timetable);
					this.ttbloading = false;

					this.updateTime = setInterval(() => {
						this.timetable = this.editTimetable(timetable);
					}, 600);

					// homeworks
					const homeworks = recap.homeworks;
					this.homeworks = this.formatHomeworks(homeworks);
					this.hwLoading = false;

					// grades
					this.grades = recap.grades.last;
					this.gradesLoading = false;

					// news
					this.news = recap.news;
					this.newsLoading = false;
				});
			},
			formatHomeworks(homeworks) {
				let homeworkDays = [];
				let today = new Date();

				// sort homeworks by day
				for (let i = 0; i < homeworks.length; i++) {
					let homework = homeworks[i];
					let date = new Date(homework.data.date);

					homeworks[i].homework.content = homeworks[i].homework.content.replace('<br/>', ' ');

					homeworks[i].data.timeLeft = Math.floor((date - today) / 1000 / 60 / 60 / 24);

					let day = homeworkDays.find((day) => {
						return day.date == date.toDateString();
					});

					if (!day) {
						day = {
							date: date.toDateString(),
							homeworks: []
						}
						homeworkDays.push(day);
					}

					day.homeworks.push(homework);
				}

				// sort homeworkDays by date
				homeworkDays.sort((a, b) => {
					return new Date(a.date) - new Date(b.date);
				});

				return homeworkDays;
			},
			reorder() {
				let order = ["comp-hw", "comp-tt"]

				let components = document.getElementById("components");
				if (components) {
					for (let i = 0; i < order.length; i++) {
						let comp = document.getElementById(order[i]);
						if (comp) {
							components.appendChild(comp);
						}
					}
				}
			},
			checkUndone() {
				// get number of undone homeworks (for badge)
				let homeworkDays = this.homeworks;

				let tomorrowDate = new Date(this.$rn);
				tomorrowDate.setDate(tomorrowDate.getDate() + 1);

				let tomorrow = homeworkDays.find((day) => {
					return day.date == tomorrowDate.toDateString();
				});

				// for each homework, check if it's done
				let undone = 0;
				if (tomorrow) {
					tomorrow.homeworks.forEach((homework) => {
						if (!homework.data.done) {
							undone++;

							NotificationBadge.setBadgeCount({
								count: undone,
							})

						}
					});
				}
			},
			async handleRefresh(event) {
				this.homeworks = [];
				this.grades = [];

				this.getRecap(true);

				this.noCoursesMsg = this.randomMsg();
				this.noCoursesEmoji = this.randomEmoji();

				this.connected = await Network.getStatus()
				this.connected = this.connected.connected;

				setTimeout(() => {
					event.detail.complete();
				}, 1000);
			},
			async displayBetaMsg() {
				const alert = await alertController.create({
					header: 'Page d\'accueil',
					message: 'Cette page est en cours de développement. Elle ne contient pas tout le contenu prévu et subira de multiples changements à l\'avenir.',
					mode: 'md',
					buttons: ['Je comprends']
				});

				await alert.present();
			},
			async getAvatar() {
				if(localStorage.getItem('customAvatar')) {
					this.avatar = localStorage.getItem('customAvatar');
				}
				else if(localStorage.getItem('avatarCache')) {
					this.avatar = localStorage.getItem('avatarCache');
				}

				document.addEventListener('userDataUpdated', () => {
					if(localStorage.getItem('customAvatar')) {
						this.avatar = localStorage.getItem('customAvatar');
					}
					else if(localStorage.getItem('avatarCache')) {
						this.avatar = localStorage.getItem('avatarCache');
					}
				});
			}
		},
		async mounted() {
			if (localStorage.getItem('userData')) {
				// get first name
				let name = JSON.parse(localStorage.getItem('userData')).student.name;
				// get last word of name
				this.firstName = name.split(' ').pop();
			}

			this.connected = await Network.getStatus()
			this.connected = this.connected.connected;

			// get data
			this.getRecap();
			this.getAvatar();

			document.addEventListener('tokenUpdated', () => {
				this.getRecap();
			});

			// get userData
			this.userData = JSON.parse(localStorage.userData);

			this.userName = JSON.parse(localStorage.userData).student.name.split(" ")[JSON.parse(localStorage.userData).student.name.split(" ").length - 1]

			// check internet connection
			Network.addListener('networkStatusChange', (status) => {
				this.internetConnection = status.connected;
			});

			// check if user already disconnected
			if(!navigator.onLine) {
				this.internetConnection = false;
			}

			// check toolbar color
			if(localStorage.getItem('fillToolbar') == 'true') {
				this.toolbarColor = 'primary';
				StatusBar.setStyle({style: Style.Dark})
			}
			else {
				this.toolbarColor = '';
			}
			
			document.addEventListener('settingsUpdated', () => {
				if(localStorage.getItem('fillToolbar') == 'true') {
					this.toolbarColor = 'primary';
				}
				else {
					this.toolbarColor = '';
				}
			});
		}
	});
</script>

<template>
	<ion-page ref="page">
		<IonHeader class="AppHeader">
			<IonToolbar class="toolbar" :color="toolbarColor">
				<ion-buttons slot="start">
					<ion-menu-button mode="md"></ion-menu-button>
				</ion-buttons>

				<ion-title mode="md">Vue d'ensemble</ion-title>

				<ion-buttons slot="end">
					<ion-nav-link v-if="avatar" router-direction="forward" :component="UserView">
						<ion-avatar class="userAvatar">
							<img :src="avatar"/>
						</ion-avatar>
					</ion-nav-link>
				</ion-buttons>
			</IonToolbar>

			<IonToolbar class="nextToolbar toolbar" :color="toolbarColor">
				
			</IonToolbar>
		</IonHeader>

		<ion-list id="comp-tt" class="nextCourse" ref="comp-tt" lines="none">
			<div class="coursElemNext" v-for="cours in timetable" :key="cours.id" :style="`--courseColor: ${cours.course.color};`">
				<ion-item class="nextCours" button :detail="false" mode="md" lines="none"
					@click="goto('timetable')" 
					:class="{ cancelled: cours.status.isCancelled, HasStatus: cours.hasStatus }">
					<div slot="start" class="timeChip">
						<IonChip>
							{{ cours.time.start.toLocaleString('fr-FR', { hour: '2-digit', minute: '2-digit' }) }}
						</IonChip>
					</div>
					<ion-label :style="`--courseColor: ${cours.course.color};`">
						<h2><span class="courseColor"></span>{{ cours.data.subject }}</h2>

						<div class="progression" v-if="nextCoursStarted">
							<p class="startProg">{{ nextCoursTime }}</p>

							<div class="progressBar">
								<div class="progress" :style="`width: ${nextCoursCompletion}%`"></div>
							</div>

									<p class="endProg">{{ cours.time.end.toLocaleString('fr-FR', { hour: '2-digit', minute: '2-digit' }) }}</p>
								</div>
								<div v-else>
									<p>{{ nextCoursTime }}</p>
								</div>

						<div class="CoursInfoContainer">
							<div class="CoursInfo room">
								<span class="material-symbols-outlined smol" slot="start">location_on</span>

								<p>{{ cours.data.rooms.join(', ') || 'Pas de salle' }}</p>
							</div>
							<div class="separator"></div>
							<div class="CoursInfo">
								<span class="material-symbols-outlined smol" slot="start">face</span>

								<p>{{ cours.data.teachers.join(', ') || 'Pas de professeur' }}</p>
							</div>
						</div>

					</ion-label>
				</ion-item>
				<div class="nextStatus" v-if="cours.status.status" :class="{ 'cancelled' : cours.status.isCancelled }">
					<span class="material-symbols-outlined mdls">info</span>
					<p>{{ cours.status.status }}</p>
				</div>
			</div>

			<ion-item v-if="timetable.error == 'ERR_NETWORK' && timetable.length == 0 && !connected" style="margin-top: 12px;"
				class="nextCours" button :detail="false" mode="md" lines="none">
				<div slot="start" style="margin-left: 5px; margin-right: 20px;">
					<span class="material-symbols-outlined mdls">wifi_off</span>
				</div>
				<ion-label class="ion-text-wrap">
					<h2>Aucune connexion internet</h2>
					<p>Les cours ne peuvent pas être chargés sans connection internet, réessayer plus tard...</p>
				</ion-label>
			</ion-item>

			<ion-item v-if="timetable.error == 'ERR_NETWORK' && timetable.length == 0 && connected" style="margin-top: 12px;"
				class="nextCours" button :detail="false" mode="md" lines="none">
				<div slot="start" style="margin-left: 5px; margin-right: 20px;">
					<span class="material-symbols-outlined mdls">crisis_alert</span>
				</div>
				<ion-label class="ion-text-wrap">
					<h2>Serveurs indisponibles</h2>
					<p>Les cours ne peuvent pas être chargés, nos serveurs seront bientôt de nouveaux disponibles...</p>
				</ion-label>
			</ion-item>

			<ion-item class="nextCours" button :detail="false" mode="md" v-if="!ttbLoading && timetable.length == 0" style="margin-top: 12px;"
				@click="goto('timetable')">
				<div slot="start" class="emoji">
					{{ noCoursesEmoji }}
				</div>
				<ion-label class="ion-text-wrap">
					<h2>Aucun cours</h2>
					<p>{{ noCoursesMsg }}</p>
				</ion-label>
			</ion-item>

			<ion-item v-if="ttbLoading && timetable.length == 0" class="nextCours" button :detail="false" mode="md" lines="none">
				<div slot="start" style="margin-left: 5px; margin-right: 20px;">
					<IonSpinner></IonSpinner>
				</div>
				<ion-label>
					<h2 style="display: flex;"><ion-skeleton-text class="courseColor" :animated="true" style="width: 10px;"></ion-skeleton-text> <ion-skeleton-text :animated="true" style="width: 40%;"></ion-skeleton-text></h2>
					<h3><ion-skeleton-text :animated="true" style="width: 35%;"></ion-skeleton-text></h3>
					<p><ion-skeleton-text :animated="true" style="width: 80%;"></ion-skeleton-text></p>
				</ion-label>
			</ion-item>
		</ion-list>

		<ion-content :fullscreen="true">
			<ion-refresher slot="fixed" @ionRefresh="handleRefresh($event)">
				<ion-refresher-content></ion-refresher-content>
			</ion-refresher>

			<Transition name="NoCoursAnim">
				<div class="NoCours" v-if="!internetConnection">
					<h1>🌏</h1>
					<h2>Pas de connexion à Internet</h2>
					<p>Vous pouvez uniquement consulter les journées déjà chargées préalablement lorsque vous êtes hors-ligne.</p>
				</div>
				<div class="NoCours" v-else-if="!allLoaded && showLoading">
					<ion-spinner></ion-spinner>
					<h2>Chargement de vos données</h2>
					<p>Nous sommes en train de récupérer vos données depuis votre service scolaire.</p>
				</div>
			</Transition>

			<div id="components" ref="components">
				<Transition name="ElemAnim">
					<ion-list v-if="allLoaded && !gradesLoading" id="comp-grades" ref="comp-grades" lines="none" inset="true" class="hw_group">
							<ion-list-header>
								<ion-label>
									<h2 style="font-size: 20px;">Dernières notes</h2>
								</ion-label>
								<ion-button @click="goto('grades')">Voir tout</ion-button>
							</ion-list-header>

							<div v-if="!gradesLoading">
								<IonNavLink v-for="grade in grades" :key="grade.id" router-direction="forward" :component="MarkView" :componentProps="{markID: grade.id}">
									<ion-item button>
										<ion-label :style="`--courseColor: ${grade.subject.color};`">
											<p><span class="courseColor"></span> {{ grade.subject.name }}</p>
											<h2>{{ grade.info.description }}</h2>
										</ion-label>

										<div slot="end">
											<ion-label v-if="grade.info.significant">
												<h2>{{ grade.grade.value }}<small>/{{ grade.grade.out_of }}</small></h2>
											</ion-label>
											<ion-label v-else>
												<h2>{{ grade.info.significantReason }}</h2>
											</ion-label>
										</div>
									</ion-item>
								</IonNavLink>
							</div>
					</ion-list>
				</Transition>

				<Transition name="ElemAnim">
					<ion-list v-if="allLoaded && !hwloading" id="comp-hw" ref="comp-hw" lines="none" inset="true">
							<ion-list-header v-if="allLoaded && !hwloading">
								<ion-label>
									<h2 style="font-size: 20px;">Travail à faire</h2>
								</ion-label>
								<ion-button @click="goto('homework')">Voir tout</ion-button>
							</ion-list-header>

							<div v-if="!hwloading"><ion-item-group class="hw_group" v-for="(day, i) in homeworks" :key="i">
								<div class="homepage_divider">
									<p>{{ new Date(day.date).toLocaleString('fr-FR', { weekday: 'long' }) }}</p>
									<div class="divider"></div>
								</div>

								<IonNavLink v-for="homework in day.homeworks" :key="homework.id" router-direction="forward" :component="HomeworkItemView" :componentProps="{urlHw: encodeURIComponent(JSON.stringify(homework))}">
								<ion-item button>
									<ion-label :style="`--courseColor: ${homework.data.color};`">
										<p><span class="courseColor"></span> {{ homework.homework.subject }}</p>
										<h2>{{ homework.homework.content }}</h2>
									</ion-label>

									<ion-chip slot="end" v-if="homework.data.done" color="success">
										<span class="material-symbols-outlined mdls">check_circle</span>
										Fait
									</ion-chip>
									<ion-chip slot="end" v-else color="medium">
										<span class="material-symbols-outlined mdls">schedule</span>
										<p v-if="homework.data.timeLeft > 1">{{homework.data.timeLeft}} jours</p>
										<p v-else-if="homework.data.timeLeft > 0">1 jour</p>
										<p v-else-if="homework.data.timeLeft < 0">Aujourd'hui</p>
										<p v-else>Demain</p>
									</ion-chip>
								</ion-item></IonNavLink>
							</ion-item-group></div>

							<ion-item v-if="homeworks.length == 0 && !hwLoading" lines="none">
								<div slot="start" style="margin-left: 5px; margin-right: 20px;">
									<span class="material-symbols-outlined mdls">done_all</span>
								</div>
								<ion-label>
									<h2>Pas de devoirs</h2>
									<p>Vous n'avez aucun travail à faire pour les 7 prochains jours.</p>
								</ion-label>
							</ion-item>
					</ion-list>
				</Transition>

				<Transition name="ElemAnim">
					<ion-list v-if="allLoaded && !newsLoading" id="comp-news" ref="comp-news" lines="none" inset="true">
							<ion-list-header>
								<ion-label>
									<h2 style="font-size: 20px;">Actualités</h2>
								</ion-label>
								<ion-button @click="goto('news')">Voir tout</ion-button>
							</ion-list-header>

							<IonNavLink v-for="(info, i) in news.slice(0, 5)" :key="i" router-direction="forward" :component="InfoView" :componentProps="{urlNews: encodeURIComponent(JSON.stringify(info))}">
								<ion-item button>
									<span slot="start" class="material-symbols-outlined mdls emoji">feed</span>
										
									<ion-label>
										<h2>{{ info.title }}</h2>
										<p>{{ info.content }}</p>
									</ion-label>
								</ion-item>
							</IonNavLink>

							<ion-item v-if="news.length == 0 && !newsLoading" lines="none">
								<div slot="start" style="margin-left: 5px; margin-right: 20px;">
									<span class="material-symbols-outlined mdls">newspaper</span>
								</div>
								<ion-label>
									<h2>Aucune nouvelle actualité</h2>
									<p>Votre établissement n'a pas encore publié d'actualités.</p>
								</ion-label>
							</ion-item>
					</ion-list>
				</Transition>
			</div>
			

		</ion-content>
	</ion-page>
</template>

<style scoped>
	.iconDisplay {
		display: flex;
		align-items: center;

		opacity: 0.8;

		width: fit-content;

		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
		margin-bottom: 3px;
	}

	.emoji {
		font-size: 1.5em;
		margin-right: 20px;
	}

	.nextCourse ion-chip {
		padding: 2px 9px !important;
		height: fit-content;
		font-weight: 600;
		font-size: 16px;
		font-family: var(--papillon-font);
	}

	.nextToolbar {
		height: 40px;
		overflow: visible;
	}

	.nextToolbar .toolbar-container {
		overflow: visible;
	}

	.nextCourse {
		overflow: visible;
		background: none;
		padding: 0;

		margin-top: -40px;
		z-index: 99999;
	}

	.nextCours {
		margin-top: 0 !important;
		margin-bottom: 10px !important;
		border-radius: 10px;
		overflow: hidden !important;

		box-shadow: 
			0px 0px 1px #00000020,
			0px 1px 5px #00000010
		;
		border-top: 0.5px solid #00000010;

		margin: 0px 16px;

		--ion-item-background: #fff;
	}

	.dark .nextCours {
		--ion-item-background: var(--ion-color-step-100) !important;
	}

	.nextCours::part(native) {
		border-radius: 10px;
		overflow: hidden !important;
	}

	.courseColor {
		width: 10px;
		height: 10px;
		border-radius: 50%;
		background-color: var(--courseColor);
		display: inline-block;
		margin-right: 5px;
	}

	.nextCours .timeChip {
		margin-right: 15px;
	}

	.nextCours .timeChip ion-chip {
		font-weight: 500;
	}

	.nextCours .progression {
		display: flex;
		align-items: center;
		gap: 5px;
	}

	.nextCours .progression .progressBar {
		width: 100%;
		height: 5px;
		background-color: var(--ion-color-step-150);
		border-radius: 5px;
		margin-right: 5px;
		transition: 5s;
	}

	.nextCours .progression .progressBar .progress {
		height: 100%;
		background-color: var(--courseColor);
		border-radius: 5px;
	}

	.nextCours .progression .startProg {
		opacity: 0.5;
	}

	.nextCours .progression .endProg {
		font-weight: 500;
	}

	.nextCours.cancelled ion-chip {
		opacity: 0.5;
	}

	.nextCours.cancelled ion-label {
		opacity: 0.5;
	}

	.nextCours.HasStatus {
		border-radius: 12px 12px 0 0 !important;
	}

	.nextCours.HasStatus::part(native) {
		border-radius: 12px 12px 0 0 !important;
	}

	.nextStatus {
		width: calc(100% - 16px * 2);
		background: var(--courseColor) !important;

		display: flex;
		align-items: center;
		gap: 12px;

		margin: 0px 16px;
		padding: 8px 24px;

		margin-top: -11px;
		margin-bottom: 5px;
		color: #fff;

		border-radius: 0 0 12px 12px !important;
	}

	.nextStatus * {
		margin: 0;
	}

	.nextStatus span {
		font-size: 20px;
	}

	.nextStatus p {
		font-size: 15px;
		font-weight: 500;
	}

	.nextStatus.cancelled {
		background: #FF453A !important;
	}

	.homepage_divider {
		display: flex;
		align-items: center;
		margin: 10px 18px;
	}

	.md .homepage_divider {
		margin: 10px 16px;
	}

	.ios .homepage_divider {
		width: 100%;
	}

	.homepage_divider p {
		margin: 0;
		font-size: 1em;
		font-weight: 500;
		font-family: var(--papillon-font);
		opacity: 0.5;
	}

	.homepage_divider .divider {
		flex: 1;
		height: 1px;
		background-color: var(--ion-color-step-150);
		margin-left: 10px;
	}

	.hw_group {
		padding-bottom: 10px;
	}

	.hw_group ion-item {
		--border-width: 0;
	}

	.hw_group ion-chip {
		padding-left: 10px !important;
	}

	.hw_group ion-chip span {
		margin-right: 5px;
	}

	.welcome {
		margin-top: calc(-200px - env(safe-area-inset-top));

		background-size: cover;
		background-position: center;
	}

	.welcome_content {
		background-color: #00000020;
		backdrop-filter: blur(20px);
		-webkit-backdrop-filter: blur(20px);

		padding: 8px 20px;
		padding-top: calc(200px + env(safe-area-inset-top));

		color: #fff;
	}

	.CoursInfoContainer {
		margin-top: 2px;
		display: flex;
		align-items: center;
		gap: 10px;

		width: calc(100vw - 145px);
		margin-top: 5px;
	}

	.CoursInfoContainer .separator {
		width: 1px;
		height: 15px;
		background: var(--ion-color-step-200);
	}

	.CoursInfo {
		display: flex;
		align-items: center;

		font-size: 0.9em;
		font-weight: 400;

		opacity: 0.7;

		width: fit-content;
		max-width: 55%;

		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
		margin-bottom: 3px;
	}

	.CoursInfo span {
		margin-right: 5px;
	}

	.CoursInfo.room {
		opacity: 100%;
		font-weight: 600;
	}

	a {
		text-decoration: none !important;
	}

	ion-buttons[slot=end] {
		margin-right: 12px !important;
	}

	.userAvatar {
		height: 34px;
		width: 34px;

		border: 1px solid #00000020;
	}

	.toolbar[color=primary] {
		--ion-text-color: #fff !important;
		color: #fff !important;
	}

	.NoCoursAnim-enter-from, .NoCoursAnim-leave-to {
		height: 0px;
		margin-top: -37px;
		opacity: 0;
		transform: scale(0.9) translateY(-30px);
	}

	@media screen and (min-width: 1150px) {
		#components {
			display: grid;
			/* make 2 columns */
			grid-template-columns: repeat(2, 1fr);

			gap: 16px;
			padding: 16px;
		}

		#components ion-list {
			margin: 0;
		}
	}
</style>