<script>
	import Header from './components/Header.svelte';
	import Features from './components/Features.svelte';
	import Working from './components/Working.svelte';
	import Hightlight from './components/Hightlight.svelte';
	import Contribution from './components/Contribution.svelte';
	import Footer from './components/Footer.svelte';
	import { featuresData, working } from './landingPageData.js';
	import io from 'socket.io-client';
	import secretKeyGenerator from './utils/secretKeyGenerator';
	import sleep from './utils/sleep';
	import { slide } from 'svelte/transition';
	import { fade, fly } from 'svelte/transition';
	import { scale } from 'svelte/transition';
	import { quintOut } from 'svelte/easing';
	import copy from 'copy-to-clipboard';
	import AnimalAvatar from 'animal-avatars.js';
	import getTime from './utils/getTime';
	import notificationSound from './utils/notificationSound';
	import checkDevice from './utils/device';
	import FileSaver from 'file-saver';
	import InternetConnection from "svelte-internet-connection";
	import nacl from 'tweetnacl';
	import naclUtil from 'tweetnacl-util';
	import { onMount } from 'svelte';
    import Analytics from 'analytics'
    import googleAnalytics from '@analytics/google-analytics'
  
    const analytics = Analytics({
      app: 'chatsecure-online',
      plugins: [
        googleAnalytics({
          trackingId: 'UA-143444026-5'
        })
      ]
    })
 
    /* Track a page view */
    analytics.page()

 	let socket,
		secretKey,
		secretLink,
		isChatBox = false,
		joinKey = '',
		joinedSession = false,
		chatmessage = '',
		notification = false,
		messages = [],
		notificationMessage,
		secretKeyCopied = false,
		secretLinkCopied = false,
		isChatLocked = false,
		inputRef,
		sessionInProgress = false,
		isLoadingJoin = false,
		isLoadingStart = false,
		user = new AnimalAvatar(),
		anon = new AnimalAvatar(),
		isTyping = false,
		network = true,
		tabActive = true,
		chatArea,
		appleShare = false,
		shareOptions = false,
		chatOptions = false,
		file = false,
		fileData = null,
		uploadButtonRef = false,
		fileLargeError = false,
		sendingMessage = false;

	let userName,
	 	userAvatar,
	 	anonName,
	 	anonAvatar;

	let userPublicKey,
		userPrivateKey,
		anonPublicKey,
		anonPrivateKey,
		nonce;

	let whatsappLink,
		telegramLink,
		imessageLink,
		messengerLink,
		emailLink;

	const MAX_FILE_SIZE = 200000; //200kb

	const NOTIFICATION_VISIBLE_TIME = 2000;

	onMount( async() => {
		const urlParams = new URLSearchParams(window.location.search);
		const param = urlParams.get('sc');
		if(param){
			joinKey = param;
			joinSession();
			secretLink = `https://keychat.online/?sc=${param}`;
			whatsappLink = `https://wa.me/?text=${secretLink}`;
			telegramLink = `https://t.me/share?url=${secretLink}`;
			emailLink = `mailto:someone@yoursite.com?body=${secretLink}`
		}

		if(param) window.history.replaceState({}, document.title, "/");
		//need to figure out how to send links in imessages and facebook

		// const appleDevice = checkDevice();

		// if(appleDevice){
		// 	appleShare = true;

		// }
	});

	//connect to socket.io server
	if(!socket){
	    // socket = io.connect('http://yourdomain/', {path: '/ws/'})
	    socket = io.connect(':3001')

	    //listen for notification message
	    socket.on('botMessage', status => {
	    	let botMessage;
	    	//status 1 = joined, 0 = disconnected
	    	if(status === 1){
	    		shareOptions = false;
	    		botMessage = `${anonName} has joined the chat`;
	    		isChatLocked = true;
	    		showNotification(botMessage, 'green');
	    		notificationSound("./assets/sounds/enterleave.mp3");
	    	} else if (joinedSession){
	    		botMessage = `${userName} has left the chat`;
				isChatLocked = false;
				shareOptions = true;
    			showNotification(botMessage, 'red');
    			notificationSound("./assets/sounds/enterleave.mp3");
	    	} else{
	    		botMessage = `${anonName} has left the chat`;
				isChatLocked = false;
				shareOptions = true;
    			showNotification(botMessage, 'red');
    			notificationSound("./assets/sounds/enterleave.mp3");
	    	}
	    });
    }

	socket.on('userData', data => {
		anonName = data.anonName;
		anonAvatar = data.anonAvatar;
		userName = data.userName;
		userAvatar = data.userAvatar;
		nonce = naclUtil.decodeBase64(data.nonce);
		userPublicKey = naclUtil.decodeBase64(data.userPublicKey);
		anonPublicKey = naclUtil.decodeBase64(data.anonPublicKey);
	})

    socket.on('sendMessage', message => {
    	let payload, utf8, boxData;
    	boxData = naclUtil.decodeBase64(message);
    	if(!joinedSession){
			payload = nacl.box.open(boxData, nonce, anonPublicKey, userPrivateKey)
			utf8 = JSON.parse(naclUtil.encodeUTF8(payload));
		} else{
			payload = nacl.box.open(boxData, nonce, userPublicKey, anonPrivateKey)
			utf8 = JSON.parse(naclUtil.encodeUTF8(payload));
			
		}
		messages = [...messages, utf8];
    	updateScroll();
    	if(!tabActive){
    		notificationSound("./assets/sounds/message.mp3");
    	}
    });

	function showNotification(msg, color){
		notification = true;
		notificationMessage = {msg, color};
		setTimeout(() => notification = false, NOTIFICATION_VISIBLE_TIME);
	}

	socket.on('typing', () => {
		isTyping = true;
	});

	socket.on('noLongerTyping', () => {
		isTyping = false;
	});

	function startSession(){
		const keys = nacl.box.keyPair();
		userPublicKey = keys.publicKey;
		userPublicKey = naclUtil.encodeBase64(userPublicKey);
		userPrivateKey = keys.secretKey;
		console.log(`YOUR PUBLIC KEY BASE64: ${userPublicKey}`);
		nonce = naclUtil.encodeBase64(nacl.randomBytes(24));

		secretKey = secretKeyGenerator();
		userName = user.getAvatarName();
		anonName = anon.getAvatarName();
		let userNameSeed = userName.replace(/ /g, '');
		let anonNameSeed = anonName.replace(/ /g, '');
		// ?options[colorful]=1%options[w]=250
	 	userAvatar = `https://avatars.dicebear.com/api/gridy/${userNameSeed}.svg?options[r]=50`;
	 	anonAvatar = `https://avatars.dicebear.com/api/gridy/${anonNameSeed}.svg?options[r]=50`;
		socket.emit('newRoom', {secretKey, userName, userAvatar, anonName, anonAvatar, userPublicKey, nonce});
		isLoadingStart = true;
		setTimeout(() => {
			isLoadingStart = false;
			isChatBox = true
		}, 300) //ux

		shareOptions = true;
		secretLink = `https://keychat.online/?sc=${secretKey || joinKey}`;
		whatsappLink = `https://wa.me/?text=${secretLink}`;
		telegramLink = `https://t.me/share?url=${secretLink}`;
		emailLink = `mailto:someone@yoursite.com?body=${secretLink}`
	}

	function joinSession(){
		const keys = nacl.box.keyPair();
		anonPublicKey = keys.publicKey;
		anonPublicKey = naclUtil.encodeBase64(anonPublicKey);
		const base64PublicKey = anonPublicKey;
		anonPrivateKey = keys.secretKey;

		isLoadingJoin = true;
		socket.emit('joinRoom', {joinKey, anonPublicKey})
		socket.on('sessionLocked', () => {
			sessionInProgress = true;
		});	

		// this is a bad way to write this, need to use promise
		setTimeout(() => { 
			 if(!sessionInProgress){
				isChatBox = true;
				joinedSession = true;
				isChatLocked = true;
				console.log(`YOUR PUBLIC KEY BASE64: ${base64PublicKey}`);
			} else{
				isLoadingJoin = false;
				showNotification('Session does not exist', 'red');
				sessionInProgress = false;
			}
		}, 1500)

		secretLink = `https://keychat.online/?sc=${secretKey || joinKey}`;
		whatsappLink = `https://wa.me/?text=${secretLink}`;
		telegramLink = `https://t.me/share?url=${secretLink}`;
		emailLink = `mailto:someone@yoursite.com?body=${secretLink}`
	}

	function closeSession(){
		isChatBox = false;
		chatmessage = '';
		messages = [];
		joinKey = '';
		joinedSession = false;
		chatmessage = '';
		notification = false;
		isChatLocked = false;
		sessionInProgress = false;
		isLoadingJoin = false;
		isLoadingStart = false;
		isTyping = false;
		network = true;
		chatArea;
		chatOptions = false;
		userName = false;
		anonName = false;
		userAvatar = false;
		anonAvatar = false;

		socket.emit('disconnecting')
	}

	async function sendMessage(){
		if(!file){
			messages = [...messages, {way: 'out', msg: chatmessage, file, fileData, time: getTime()}];
		}
		let data = JSON.stringify({way: 'in', msg: chatmessage, file, fileData, time: getTime()});
		let box;
		if(joinedSession){
			box = nacl.box(
  			naclUtil.decodeUTF8(data),
  			nonce,
  			userPublicKey,
  			anonPrivateKey)
		} else{
			box = nacl.box(
  			naclUtil.decodeUTF8(data),
  			nonce,
  			anonPublicKey,
  			userPrivateKey)	
		}
		box = naclUtil.encodeBase64(box);
		socket.emit('message', box);
		if(file){
			sendingMessage = true;						
			await sleep(3.8);
			sendingMessage = false;
			messages = [...messages, {way: 'out', msg: chatmessage, file, fileData, time: getTime()}];
		}
		chatmessage = '';
		file = false;
		fileData = null;
		inputRef.focus()
		updateScroll();
	}

	function copySecretKey(){
		secretKeyCopied = !secretKeyCopied;
		if (secretKey){
			copy(secretKey)
		} else {
			copy(joinKey)
		} 
		showNotification("Secret Key copied", 'green')
		setTimeout(() => secretKeyCopied = false, NOTIFICATION_VISIBLE_TIME);
	}

	function copySecretLink(){
		secretLinkCopied = !secretLinkCopied;
		copy(secretLink)
		showNotification("Secret Link copied", 'green')
		setTimeout(() => secretLinkCopied = false, NOTIFICATION_VISIBLE_TIME);
	}

	let typing = false, 
		timeout = undefined;

	function timeoutFunction(){
		typing = false;
		socket.emit('noLongerTypingMessage');
	}

	async function saveHistory(){
		let chatHistory = messages;
		chatHistory = chatHistory.map(message => ({
			way: message.way,
			message: message.msg,
			time: message.time
		}));
		const data = JSON.stringify(chatHistory);
		const blob = await new Blob([data], {type: "text/plain;charset=utf-8"});
		FileSaver.saveAs(blob, `${secretKey || joinKey}.txt`);
		chatOptions = false;
		chatmessage = '';
	}


	 $: if (chatmessage === "/" || chatmessage[chatmessage.length-1] == "/"){
		chatOptions = true;	 	
	 } else{
	 	chatOptions = false;
	 }

 	// $: if(!network && isChatBox){
		// showNotification('You are offline', 'red');
 	// }
 	// else if (network && isChatBox && messages.length){
 	// 	showNotification('You are back online', 'green');
 	// }

	function checkEnterPress(event){
		let key;
		let keyCode;
		key = event.key;
		keyCode = event.keyCode;

		if(keyCode === 13 && !isChatBox && !joinKey.length && !chatmessage.length){
			startSession();
		} else if(keyCode === 13 && joinKey.length && !isChatBox && !chatmessage.length){
			joinSession();
		}

		if(keyCode === 13 && chatmessage.length && isChatBox){
			sendMessage();
		}
		
		if(typing === false && chatmessage) {
	    	typing = true;
	    	socket.emit('typingMessage');
	    	timeout = setTimeout(timeoutFunction, 200);
	  	} else if(typing === true && chatmessage){
	    	clearTimeout(timeout);
	    	timeout = setTimeout(timeoutFunction, 200);
	  	}
	}

	function updateScroll() {
		setTimeout(() => {
    		chatArea.scrollTop = chatArea.scrollHeight;  
  		}, 0);
	}

	function uploadFile(e){
		let fileRef = e.target.files[0];
		if (fileRef.size > MAX_FILE_SIZE){
			fileLargeError = true;
			setTimeout(()=>{
				fileLargeError = false
			}, NOTIFICATION_VISIBLE_TIME)
			return;
		}
		chatmessage = fileRef.name;
		file = true;
		fileData = fileRef;
		const reader = new FileReader();
 		reader.readAsDataURL(fileData); 
 		reader.onloadend  = function() {
     		 fileData = reader.result;                
 		}
	}	

	function cancelUpload(){
		chatmessage = '';
		file = false;
		fileData = null;
	}
</script>

<svelte:head>
	<link rel="icon" type="image/png" href={!isChatBox ? "assets/offlinefav.svg" : "assets/onlinefav.svg"}>
</svelte:head>

<svelte:body
	on:mouseenter={()=>tabActive = true}
	on:mouseleave={()=>tabActive = false}
/>
<!-- causing a lot of bugs, will fix -->
<!-- <svelte:window on:keydown={checkEnterPress}/> -->
<div class="main-container" class:modifier={isChatBox}>
<div class:modifier--displayNone={isChatBox}>
<Header src={isChatBox ? "assets/chatsecureonline.svg" : "assets/chatsecureoffline.svg"} />
</div>
<main class:main--modifier={isChatBox}>
	<div class="enterSessionCard" class:enterSessionCard--modifier={isChatBox}>
		{#if !isChatBox}
			<input placeholder="*****" minlength="5" maxlength="5" bind:value={joinKey} on:keydown={checkEnterPress}>
			{#if notification}
				<div transition:slide class="error" class:redtext={notificationMessage.color === 'red'}>
					{notificationMessage.msg}
				</div>
			{/if}
			<button on:click={joinSession} style="background: var(--grey)" disabled={!joinKey.length} class:focus={joinKey.length}>
				{#if !isLoadingJoin}
					Enter with secret code
				{:else}
					<img src="assets/loader.gif" alt="loading gif" class="loaderAnim">
				{/if}
			</button>
		{:else}
		<!-- this is just here to laod the image, avoid the delay when rendering in the real case -->
		<div style="background-image: url({userAvatar})"></div>
		<div style="background-image: url({anonAvatar})"></div>
			<div class="chatBox" transition:slide>
				<div class="sessionInfo flex" in:fade={{duration: 500}}>
					<div class="secretKey">
						<div data-tooltip="Share this secret code to start chatting">{secretKey || joinKey}</div>
						<div class="flex">
							<img src={secretKeyCopied ? "assets/copied.svg" : "assets/copy.svg"} alt="copied icon" class="copyIcon" on:click={copySecretKey}>
							<img src={!shareOptions ? "assets/share.svg" : "assets/shareactive.svg"} alt="share icon" class="copyIcon" on:click={()=>{shareOptions = !shareOptions}} style="margin-left: 0.4rem; margin-right: -0.2rem;">
						</div>
					</div>
					<ul>
						<li data-tooltip="Locked once another user enters the chat" style="cursor: pointer;">
							<div class={isChatLocked ? 'status' : joinedSession ? 'status' : 'status red'}></div>
							Chat {isChatLocked ? 'Locked' : joinedSession ? 'Locked' : 'Open'}
						</li>
						<li data-tooltip="Indicates if you are online or not" style="cursor: pointer;">
							<InternetConnection let:status>
								<div class="status" class:red={status === 'offline'}></div>
							</InternetConnection>
							Network
						</li>
					</ul>
				</div>
				{#if shareOptions}
					<div class="shareOptions" in:fly="{{y: -20, duration: 100}}" out:fly="{{ y: -20, duration: 100 }}">
						<h5 style="display: inline-block;">Share link</h5>
						<img src="assets/close.svg" alt="close icon" class="closeIcon" on:click={()=>shareOptions = false}>
						<div class="secretKey" style="padding: 0 0.4rem;">
							<div class="sharelink" data-tooltip="Share this link to start chatting">{secretLink}</div>
							<img src={secretLinkCopied ? "assets/copied.svg" : "assets/copy.svg"} alt="copied icon" class="copyIcon" on:click={copySecretLink} style="margin-left: auto;">
						</div>
						<div>
							<a href={whatsappLink} target="_blank">
								<img src="assets/whatsapp.svg" title="WhatsApp" alt="whatsapp icon" class="chatOption">
							</a>
							<a href={telegramLink} target="_blank">
								<img src="assets/telegram.svg" title="Telegram" alt="Telegram" class="chatOption"></a>
							<!-- {#if appleShare}
								<a href={imessageLink} target="_blank">
									<img src="assets/imessage.svg" title="iMessage" alt="iMessage" class="chatOption">
								</a>
							{/if}
							<a href={messengerLink} target="_blank">
								<img src="assets/messenger.svg" title="Facebook Messenger" alt="messenger icon icon" class="chatOption">
							</a> -->
							<a href={emailLink} target="_blank">
								<img src="assets/email.svg" title="Email" alt="email icon" class="chatOption">
							</a>
						</div>
					</div>
				{/if}
				{#if notification}
					<div class="notification" in:fly="{{y: -20, duration: 200}}" out:fly="{{ y: -20, duration: 200 }}" class:red={notificationMessage.color === 'red'}>
						{notificationMessage.msg}
					</div>
				{/if}
				<div class="chatArea" in:fade={{duration: 500}} bind:this={chatArea}>
					{#each messages as {way, msg, time, file, fileData}}
						{#if way === 'out'}
							<div class="chatBubbleContainer" in:fly="{{y: 10, duration: 300}}">
								<div class="chatBubbleBlue">
									{#if file}
										<a download={msg} href={fileData} style="word-wrap: anywhere;">
											{msg}<img src="assets/attachment.svg" alt="attachment icon">
										</a>
									{:else}
										{msg}
									{/if}
								</div>
								{#if !joinedSession}
									<div class="avatar avatarUser tooltip-bottom-right" style="background-image: url({userAvatar}); font-size: 0.8rem;" data-tooltip={userName}></div>
								{:else}
									<div class="avatar avatarUser tooltip-bottom-right" style="background-image: url({anonAvatar}); font-size: 0.8rem;" data-tooltip={anonName}></div>
								{/if}
							</div>
							<div class="timeStamp timeStampUser" in:fly="{{y: 10, duration: 300}}">{time}</div>
						{:else}
							<div class="chatBubbleContainer" in:fly="{{y: 10, duration: 300}}">
								{#if !joinedSession}
									<div class="avatar avatarAnon tooltip-bottom-left" style="background-image: url({anonAvatar}); font-size: 0.8rem;" data-tooltip={anonName}></div>
								{:else}
									<div class="avatar avatarAnon tooltip-bottom-left" style="background-image: url({userAvatar}); font-size: 0.8rem;" data-tooltip={userName}></div>
								{/if}
								<div class="chatBubbleGrey">
									{#if file}
										<a download={msg} href={fileData} style="word-wrap: anywhere;">
											<img src="assets/attachment.svg" alt="attachment icon" style="filter: brightness(10);">{msg}
										</a>
									{:else}
										{msg}
									{/if}
								</div>						
							</div>
							<div class="timeStamp timeStampAnon" in:fly="{{y: 10, duration: 300}}">{time}</div>
						{/if}
					{/each}
					{#if isTyping}
						<div class="chatBubbleContainer" in:fly="{{y: 10, duration: 300}}" out:fly="{{y: 10, duration: 100}}">
								{#if !joinedSession}
									<div class="avatar avatarAnon tooltip-bottom-left" style="background-image: url({anonAvatar}); font-size: 0.8rem;" data-tooltip={anonName}></div>
								{:else}
									<div class="avatar avatarAnon tooltip-bottom-left" style="background-image: url({userAvatar}); font-size: 0.8rem;" data-tooltip={userName}></div>
								{/if}
							<div class="chatBubbleGrey typingAnimContainer"><img src="assets/typing.gif" alt="typing anim" class="typingAnim"></div>
						</div>
					{/if}
				</div>
				{#if chatOptions}
					<ul class="chatOptions" transition:slide>
						<li on:click={closeSession}><img src="assets/close.png" alt="close icon"><div>close</div></li>
						<li on:click={()=>{messages = []; chatOptions = false; chatmessage = '';}}><img src="assets/clear.png" alt="clear icon"><div>clear</div></li>
						<li on:click={saveHistory}><img src="assets/download.png" alt="download icon"><div>download</div></li>
					</ul>
				{/if}	
				{#if !isChatLocked}	
					<div class="loadingChat" transition:slide data-tooltip="Messages cannot be sent unless key exchange happens">
						<img src="assets/loader.gif" alt="loading animation" class="loading">
						<p>waiting for a user to join...</p>
					</div>
				{/if}
				<div class="messageBoxContainer flex" in:fade={{duration: 500}}>
					<div>
						<input type='file' on:change={uploadFile} bind:this={uploadButtonRef} hidden>
						{#if file}
							<img src="assets/close.svg" alt="attachment icon" class="attachmentIcon" style="opacity: 0.2;" on:click={cancelUpload}>
						{:else}
							<img src="assets/attachment.svg" alt="attachment icon" class="attachmentIcon" class:attachmentDisabled={chatmessage.length || !isChatLocked} on:click={()=>uploadButtonRef.click()}>
						{/if}
						<input class="messageBox" bind:value={chatmessage} maxlength="1024" minlength="1" placeholder="type your message here" bind:this={inputRef}  on:keydown={checkEnterPress} disabled={!isChatLocked || file} class:fileUploaded={file} />
					</div>
					<button on:click={sendMessage} disabled={!chatmessage.length || chatmessage === "/" || sendingMessage}>
						{#if sendingMessage}
							<img src="assets/loader.gif" alt="loading gif" class="loaderAnim" style="height: 1.8rem">
						{:else}
							<img src="assets/send.png" alt="send icon" class="sendIcon">
							<span class="sendText">Send</span>
						{/if}
					</button>
				</div>
				{#if fileLargeError}	
					<p transition:slide class="fileLargeError">
						Please upload a file less than {MAX_FILE_SIZE/1000}kb
					</p>
				{/if}
				{#if !chatOptions}
					<p class="chatOptionsHelper" transition:slide>type '/' for chat options<span style="margin-left:0.8rem;" class="onlyDesktop">open console for your public key</span></p>
				{/if}
				<img src="assets/chatsecureoffline.svg" alt="logo" class="chatBoxLogo">
			</div>
		{/if}
	</div>
	{#if !isChatBox}
		<button class="newSession" on:click={startSession}>
			{#if !isLoadingStart}
				Start a new chat
			{:else}
				<img src="assets/loader.gif" alt="loading gif" class="loaderAnim">
			{/if}
		</button>
<!-- 	{:else}
		<p class="warning" in:fade class:modifier--displayNone={isChatBox}>Please do not share personal information with anyone</p> -->
	<Contribution/>
	{/if}

</main>
<div class:mobileChatActive={isChatBox}>
	<Features features={featuresData} />
</div>
<div class:mobileChatActive={isChatBox}>
	<Hightlight />
</div>
<div class:mobileChatActive={isChatBox}>
	<Working steps={working} />
</div>
</div>
<div class:mobileChatActive={isChatBox}>
	<Footer />
</div>

<style>
	main{
		width: auto;
		max-width: 24rem;
		margin: auto;
		margin-top: 0.8rem;
	}

	.main-container{
		margin: 0 1rem;
	}

	.chatBubbleContainer{
		display: flex;
		justify-content: center;
		align-items: inherit;
	}

	.typingAnimContainer{
		padding: 0 !important;
		height: 2.2rem !important;
		width: 3.4rem !important;
		display: flex !important;
		justify-content: center !important;
		align-items: center !important;
	}

	.typingAnim{
		height: 1.46rem !important;
		margin-right: 0 !important;
	}

	button:disabled,
	button[disabled]{
  		background-color: #3c3c3c;
  		pointer-events: none;
	}

	.enterSessionCard{
		padding: 1.2rem;
		height: auto;
		background: red;
		background: var(--grey);
		border-radius: 0.2rem;
		border: 1px solid var(--border-grey);
		display: flex;
		flex-direction: column;
	}

	.newSession{
		margin-top: 1rem;
	}

	.chatBox{
		position: relative;
		height: auto;
	}

/*	.alias{
		font-size: 0.6rem;
		opacity: 0.5;
		margin: -0.2rem 0;
    	margin-bottom: -0.2rem;
		margin-bottom: 0.1rem;
		text-align: right;
	}*/

	.focus{
		background: #6976f7 !important;
	}

	.loadingChat{
		display: flex;
		align-items: center;
		margin-bottom: 1rem;
	}

	.loadingChat p{
		font-size: 0.8rem;
		margin-left: 0.3rem;
		font-weight: 400;
	}

	.loading{
		height: 1.2rem;
	}

	.avatar{
		width: 2.3rem;
		height: 2.3rem;
		border-radius: 50%;
		background-repeat: no-repeat;
		background-position: center;
		background-size: cover;
		
	}

	.attachmentIcon{
		height: 1.1rem;
		right: 12px;
		position: absolute;
		top: 12px;
	}

	.fileUploaded{
		color: #7b8aff;
	}

	.attachmentDisabled{
		pointer-events: none;
		filter: grayscale(100%);
	}

	.avatarUser{
		margin-left: 0.6rem;
	}

	.avatarAnon{
		margin-right: 0.6rem;
	}

/*	.warning{
		font-weight: 300;
		text-align: center;
		font-size: 0.7rem;
		margin-top: 1.2rem;
		opacity: 0.8;
	}*/

	.sharelink{
		font-size: 0.8rem;
		opacity: 0.8;
		margin: 0 !important;
	}

	.messageBoxContainer{
		display: flex;
	}

	.timeStamp{
		font-weight: 300;
		font-size: 0.6rem;
		margin-top: -0.8em;
		margin-bottom: 1.8rem;
		opacity: 0.8;
	}

	.timeStampUser{
		text-align: right;
		margin-right: 2.8rem;
	}

	.timeStampAnon{
		text-align: left;
		margin-left: 2.9rem;
	}

	.messageBoxContainer button{
		height: 2.6rem;
		width: 28%;
	}

	.messageBoxContainer div{
		position: relative;
		width: 68%;
	}

	.onlyDesktop{
		display: inline-block;
	}

	.messageBox{
		font-size: 1rem;
		text-align: left;
		padding: 1rem;
		margin-bottom: 0;
		width: 100%;
		/*font correction*/
		padding-top: 1.1rem;
		padding-right: 2.2rem;
	}

	.sendIcon{
		height: 0.8rem;
		margin-right: 0.4rem;
	}

	.sendText{
		margin-top: 0.2rem;
	}

	.chatArea{
		height: 42vH;
		background: var(--bg-color);
		margin: 1.8rem 0;
		border-radius: 0.2rem;
		margin-top: 1.2rem;
		position: relative;
		padding: 1rem;
		text-align: center;
		overflow-y: scroll;
		margin-bottom: 1rem;
	}

	.closeIcon{
		float: right;
		height: 0.8rem;
		opacity: 0.8;
	}

	.secretKey{
		background: #181818;
		height: 2.8rem;
		padding: 0 0.2rem;
		display: inline-block;
		border-radius: 0.2rem;
		font-weight: 400;
		display: flex;
		align-items: center;
		margin: 0.8rem 0;
	}

	.sessionInfo ul{
		display: flex;
		font-weight: 400;
		font-size: 0.8rem;
	}

	.sessionInfo li{
		display: flex;
		align-items: center;
		margin-left: 0.4rem;
	}

	.status{
		min-width: 0.6rem;
		min-height: 0.6rem;
		border-radius: 50%;
		background: var(--green);
		margin-right: 0.4rem;
		/*font correction*/
		margin-bottom: 0.1rem;
	}

	.chatOptions{
		margin-bottom: 0.8rem;
		display: flex;
		align-items: center;
		border-radius: 0.2rem;
	}

	.chatOption{
	 	height: 2.6rem;
	 	margin-right: 0.2rem;
	}

	.chatOptions li{
		list-style: none;
		font-weight: 300;
		font-size: 1rem;
		background: #242424;
		padding: 0.6rem 0.8rem;
		border-radius: 0.2rem;
		cursor: pointer;
		display: flex;
		align-items: center;
		margin-right: 0.8rem;
	}

	.chatOptions li div{
		padding-top: 0.2rem;
	}

	.chatOptions img{
		height: 1rem;
		margin-right: 0.4rem
	}

	.fileLargeError{
		font-size: 0.8rem;
		margin-left: 1rem;
		margin-top: 0.6rem;
		color: var(--red);
	}

	.chatOptionsHelper{
		font-weight: 600;
		margin-left: 1rem;
		margin-top: 0.8rem;
		opacity: 0.6;
		font-size: 0.6rem;
	}

	.chatBubbleBlue{
		background: #3F8BFE;
		padding: 0.6rem;
		border-radius: 1rem 1rem 0 1rem;
		font-size: 0.8rem;
		font-weight: 400;
		margin-left: auto;
		margin-bottom: 0.8rem;
		max-width: 12rem;
		height: auto;
		word-wrap: break-word;
		text-align: right;
	}

	.chatBubbleBlue a, .chatBubbleGrey a{
		display: flex;
		align-items: center;
	}

	.chatBubbleBlue img{
		height: 0.96rem;
		filter: brightness(10);
		margin-left: 0.4rem;
	}

	.shareOptions{
		position: absolute;
		left: 1rem;
		right: 1rem;
		z-index: 1;
		background: var(--grey);
		border-radius: 0.4rem;
		padding: 1rem;
		text-align: left;
		max-width: 20rem;
		margin: auto;
		top: 6.6rem;
	}

	.chatBubbleGrey{
		background: #414141;
		padding: 0.6rem;
		border-radius: 1rem 1rem 1rem 0;
		font-size: 0.8rem;
		font-weight: 400;
		margin-right: auto;
		margin-bottom: 0.8rem;
		text-align:left;
		max-width: 12rem;
		height: auto;
		word-wrap: break-word;
		text-align: left;
	}

	.chatBubbleGrey img{
		height: 0.96rem;
		margin-right: 0.4rem;
	}

	.notification{
	    background: var(--green);
	    display: inline-block;
	    padding: 0.2rem 0.8rem;
	    border-radius: 4rem;
	    color: black;
	    font-size: 0.8rem;
	    font-weight: 600;
	    position: absolute;
	    left: 0;
	    right: 0;
	    text-align: center;
	    margin: 0 4rem;
	    z-index: 99;
	    top: 6.6rem;
	}

	.loaderAnim{
		height: 2.4rem;
	}

	.copyIcon{
		height: 2rem;
	}

	.error{
		text-align: center;
		margin-bottom: 1rem;
		font-weight: 400;
		font-size: 1rem;
		color: red;
	}

	.secretKey div{
		margin: 0.4rem;
	}

	.red{
		background: var(--red);
		color: white !important;
	}

	.redtext{
		color: var(--red);
	}

	.chatBoxLogo{
		display: none;
	}

	@media only screen and (max-width: 768px) {
		main{
			overflow-x: hidden;
		}

		.main--modifier{
			margin: 0;
			max-width: inherit;
		}

		.chatBoxLogo{
			display: block;
			height: 1.2rem;
			float: right;
			filter: grayscale(100%);
			opacity: 0.6;
			margin-top: 0.4rem;
		}

		.mobileChatActive{
			display: none;
		}

		.onlyDesktop{
			display: none;
		}

		.modifier{
			margin: 0;
		}

		.chatArea{
			height: 66vH;
		}

		.enterSessionCard--modifier{
			height: 100%;
			border: none;
		}

		.modifier--displayNone{
			display: none;
		}

		.messageBoxContainer div{
			position: relative;
			width: 76%;
		}

		.messageBoxContainer button{
			width: 20%;
		}

		.sendText{
			display: none;
		}

		.sendIcon{
			margin-right: 0;
		}
  	}
</style>
