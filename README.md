<div dir="rtl">

# 🎮 تکمیل کوئست‌های دیسکورد

اسکریپتی برای تکمیل خودکار کوئست‌های دیسکورد بدون نیاز به بازی کردن یا تماشای ویدیو!

---

## ⚠️ توجه مهم

> این اسکریپت برای کوئست‌هایی که نیاز به بازی دارند، **در مرورگر کار نمی‌کند!**
> 
> برای این نوع کوئست‌ها، از [اپلیکیشن دسکتاپ دیسکورد](https://discord.com/download) استفاده کنید.

---

## 📋 نحوه استفاده

### مرحله ۱: پذیرش کوئست
به بخش **Discover → Quests** بروید و یک کوئست را قبول کنید.

### مرحله ۲: باز کردن DevTools
کلیدهای `Ctrl + Shift + I` را فشار دهید تا DevTools باز شود.

### مرحله ۳: رفتن به تب Console
روی تب **Console** کلیک کنید.

### مرحله ۴: اجرای کد
کد زیر را کپی و در کنسول پیست کنید، سپس Enter بزنید.

> 💡 **نکته:** اگر نتوانستید پیست کنید، ابتدا `allow pasting` را تایپ کرده و Enter بزنید.

<details>
<summary>برای دیدن کد کلیک کنید</summary>

```js
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata).exports.Z;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest).exports.Z;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getAllThreadsForParent).exports.Z;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getSFWDefaultChannel).exports.ZP;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue).exports.Z;
let api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;

const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"]
let quests = [...QuestsStore.quests.values()].filter(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now() && supportedTasks.find(y => Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y)))
let isApp = typeof DiscordNative !== "undefined"
if(quests.length === 0) {
	console.log("You don't have any uncompleted quests!")
} else {
	let doJob = function() {
		const quest = quests.pop()
		if(!quest) return

		const pid = Math.floor(Math.random() * 30000) + 1000
		
		const applicationId = quest.config.application.id
		const applicationName = quest.config.application.name
		const questName = quest.config.messages.questName
		const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2
		const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null)
		const secondsNeeded = taskConfig.tasks[taskName].target
		let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0

		if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
			const maxFuture = 10, speed = 7, interval = 1
			const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime()
			let completed = false
			let fn = async () => {			
				while(true) {
					const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture
					const diff = maxAllowed - secondsDone
					const timestamp = secondsDone + speed
					if(diff >= speed) {
						const res = await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}})
						completed = res.body.completed_at != null
						secondsDone = Math.min(secondsNeeded, timestamp)
					}
					
					if(timestamp >= secondsNeeded) {
						break
					}
					await new Promise(resolve => setTimeout(resolve, interval * 1000))
				}
				if(!completed) {
					await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: secondsNeeded}})
				}
				console.log("Quest completed!")
				doJob()
			}
			fn()
			console.log(`Spoofing video for ${questName}.`)
		} else if(taskName === "PLAY_ON_DESKTOP") {
			if(!isApp) {
				console.log("This no longer works in browser for non-video quests. Use the discord desktop app to complete the", questName, "quest!")
			} else {
				api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
					const appData = res.body[0]
					const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","")
					
					const fakeGame = {
						cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
						exeName,
						exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
						hidden: false,
						isLauncher: false,
						id: applicationId,
						name: appData.name,
						pid: pid,
						pidPath: [pid],
						processName: appData.name,
						start: Date.now(),
					}
					const realGames = RunningGameStore.getRunningGames()
					const fakeGames = [fakeGame]
					const realGetRunningGames = RunningGameStore.getRunningGames
					const realGetGameForPID = RunningGameStore.getGameForPID
					RunningGameStore.getRunningGames = () => fakeGames
					RunningGameStore.getGameForPID = (pid) => fakeGames.find(x => x.pid === pid)
					FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames})
					
					let fn = data => {
						let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
						console.log(`Quest progress: ${progress}/${secondsNeeded}`)
						
						if(progress >= secondsNeeded) {
							console.log("Quest completed!")
							
							RunningGameStore.getRunningGames = realGetRunningGames
							RunningGameStore.getGameForPID = realGetGameForPID
							FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
							FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
							
							doJob()
						}
					}
					FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
					console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
				})
			}
		} else if(taskName === "STREAM_ON_DESKTOP") {
			if(!isApp) {
				console.log("This no longer works in browser for non-video quests. Use the discord desktop app to complete the", questName, "quest!")
			} else {
				let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
					id: applicationId,
					pid,
					sourceName: null
				})
				
				let fn = data => {
					let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
					console.log(`Quest progress: ${progress}/${secondsNeeded}`)
					
					if(progress >= secondsNeeded) {
						console.log("Quest completed!")
						
						ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
						FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			
						doJob()
					}
				}
				FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				
				console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
				console.log("Remember that you need at least 1 other person to be in the vc!")
			}
		} else if(taskName === "PLAY_ACTIVITY") {
			const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? Object.values(GuildChannelStore.getAllGuilds()).find(x => x != null && x.VOCAL.length > 0).VOCAL[0].channel.id
			const streamKey = `call:${channelId}:1`
			
			let fn = async () => {
				console.log("Completing quest", questName, "-", quest.config.messages.questName)
				
				while(true) {
					const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}})
					const progress = res.body.progress.PLAY_ACTIVITY.value
					console.log(`Quest progress: ${progress}/${secondsNeeded}`)
					
					await new Promise(resolve => setTimeout(resolve, 20 * 1000))
					
					if(progress >= secondsNeeded) {
						await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}})
						break
					}
				}
				
				console.log("Quest completed!")
				doJob()
			}
			fn()
		}
	}
	doJob()
}
```

</details>

### مرحله ۵: دستورالعمل‌ها را دنبال کنید
- اگر کوئست شما از نوع **"بازی کردن"** یا **"تماشای ویدیو"** است → فقط صبر کنید!
- اگر کوئست شما از نوع **"استریم کردن"** است → به یک ویس چنل بروید و هر پنجره‌ای را استریم کنید.

### مرحله ۶: دریافت جایزه
پس از تکمیل، جایزه خود را claim کنید! 🎁

---

## 🔧 فعال‌سازی DevTools در اپلیکیشن دسکتاپ

اگر `Ctrl + Shift + I` کار نمی‌کند، یکی از روش‌های زیر را امتحان کنید:

### روش A: ویرایش settings.json

1. به این مسیر بروید:
   - **ویندوز:** `%USERPROFILE%\AppData\Roaming\discord\`
   - **مک:** `~/Library/Application Support/discord/`
   - **لینوکس:** `~/.config/discord/`

2. فایل `settings.json` را باز کنید و این خط را اضافه کنید:
```json
"DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true
```

3. فایل را ذخیره کرده و دیسکورد را ریستارت کنید.

### روش B: استفاده از Discord PTB
[Discord PTB](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64) را دانلود و نصب کنید.

### روش C: کلاینت‌های مود شده
از Vencord یا BetterDiscord استفاده کنید.

---

## ❓ سوالات متداول

### آیا ممکنه بن بشم؟
همیشه ریسکی وجود داره، ولی تا الان کسی بخاطر این اسکریپت یا چیزای مشابه مثل client mod ها بن نشده.

### Ctrl + Shift + I کار نمیکنه!
- از Discord PTB استفاده کنید
- یا settings.json رو ویرایش کنید (روش A)

### Ctrl + Shift + I اسکرین‌شات میگیره!
اگه از AMD Radeon استفاده می‌کنید، این کلید میانبر رو از تنظیمات AMD غیرفعال کنید.

### خطای syntax error میده!
مطمئن شید مرورگر صفحه رو ترجمه نکرده. افزونه‌های مترجم رو غیرفعال کنید.

### میتونم کوئست‌های منقضی شده رو تکمیل کنم؟
نه، امکانش نیست.

### میشه تایمر ۱۵ دقیقه‌ای رو دور زد؟
نه، زمان شروع و باقیمانده توسط سرور مدیریت میشه.

---

## 📊 پیشرفت

می‌تونید پیشرفت رو از طریق پیام‌های `Quest progress:` در تب Console یا نوار پیشرفت در بخش Quests ببینید.

---

## 📜 لایسنس

GPL-3.0

---

## 🔗 لینک اصلی

[GitHub Gist اصلی](https://gist.github.com/aamiaa/204cd9d42013ded9faf646fae7f89fbb)

---

<div align="center">

**⭐ اگه مفید بود ستاره بدید!**

</div>

</div>
