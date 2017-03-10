@NonCPS
def jsonParse(def json) {
	new groovy.json.JsonSlurperClassic().parseText(json)
}

def nodeVersion = '4.7.3'
def tiSDKVersion = '6.0.2.GA'
def androidAPILevel = '23'

def sdkSetup(sdkVersion) {
	sh 'npm install -g appcelerator'
	sh 'appc logout'
	sh 'appc config set defaultEnvironment prod'
	def sdkListOutput = ''
	lock('appc-login:895d8db1-87c2-4d96-a786-349c2ed2c04a') { // only let one login at a time for this user!
		withCredentials([usernamePassword(credentialsId: '895d8db1-87c2-4d96-a786-349c2ed2c04a', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
			sh 'appc login --username "$USER" --password "$PASS"'
		}
		sh 'appc use latest'
		sh "appc ti sdk install ${sdkVersion} -d"
		sdkListOutput = sh(returnStdout: true, script: 'appc ti sdk list -o json')
		sh 'appc logout'
	}
	echo sdkListOutput
	def sdkListJSON = jsonParse(sdkListOutput)
	// def titaniumRoot = sdkListJSON['defaultInstallLocation']
	def activeSDKVersion = sdkListJSON['activeSDK']
	return sdkListJSON['installed'][activeSDKVersion]
}

def buildAndroid(nodeVersion, tiSDKVersion, androidAPILevel) {
	return {
		node('android-sdk && android-ndk && (osx || linux)') {
			unstash 'sources'
			nodejs(nodeJSInstallationName: "node ${nodeVersion}") {
				def activeSDKPath = sdkSetup(tiSDKVersion)

				// We have to hack to make sure we pick up correct ANDROID_SDK/NDK values from the node that's currently running this section of the build.
				def androidSDK = env.ANDROID_SDK // default to what's in env (may have come from jenkins env vars set on initial node)
				def androidNDK = env.ANDROID_NDK
				withEnv(['ANDROID_SDK=', 'ANDROID_NDK=']) {
					try {
						androidSDK = sh(returnStdout: true, script: 'printenv ANDROID_SDK')
					} catch (e) {
						// squash, env var not set at OS-level
					}
					try {
						androidNDK = sh(returnStdout: true, script: 'printenv ANDROID_NDK')
					} catch (e) {
						// squash, env var not set at OS-level
					}
					sh "appc ti config android.sdkPath ${androidSDK}"
					sh "appc ti config android.ndkPath ${androidNDK}"
					dir('android') {
						writeFile file: 'build.properties', text: """
titanium.platform=${activeSDKPath}/android
android.platform=${androidSDK}/platforms/android-${androidAPILevel}
google.apis=${androidSDK}/add-ons/addon-google_apis-google-${androidAPILevel}
"""
						// TODO Use 'appc ti build --build-only'!
						// if lib folder doesn't exist, create it
						sh 'mkdir -p lib'
						// if build folder doesn't exist, create it
						sh 'mkdir -p build'
						// if build/docs folder doesn't exist, create it
						sh 'mkdir -p build/docs'
						lock('appc-login:895d8db1-87c2-4d96-a786-349c2ed2c04a') { // only let one login at a time for this user!
							withCredentials([usernamePassword(credentialsId: '895d8db1-87c2-4d96-a786-349c2ed2c04a', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
								sh 'appc login --username "$USER" --password "$PASS"'
							}
							sh 'appc ti build -p android --build-only'
							sh 'appc logout'
						}
						dir('dist') {
							archiveArtifacts '*.zip'
						}
					} // dir
				} // withEnv
			} // nodeJs
		} // node
	}
}

def buildIOS(nodeVersion, tiSDKVersion) {
	return {
		node('osx && xcode') {
			unstash 'sources'
			nodejs(nodeJSInstallationName: "node ${nodeVersion}") {
				def activeSDKPath = sdkSetup(tiSDKVersion)

				dir('ios') {
					writeFile file: 'titanium.xcconfig', text: """
TITANIUM_SDK = ${activeSDKPath}
TITANIUM_BASE_SDK = \"\$(TITANIUM_SDK)/iphone/include\"
TITANIUM_BASE_SDK2 = \"\$(TITANIUM_SDK)/iphone/include/TiCore\"
TITANIUM_BASE_SDK3 = \"\$(TITANIUM_SDK)/iphone/include/ASI\"
TITANIUM_BASE_SDK4 = \"\$(TITANIUM_SDK)/iphone/include/APSHTTPClient\"
HEADER_SEARCH_PATHS= \$(TITANIUM_BASE_SDK) \$(TITANIUM_BASE_SDK2) \$(TITANIUM_BASE_SDK3) \$(TITANIUM_BASE_SDK4) \${PROJECT_DIR}/**
"""
					lock('appc-login:895d8db1-87c2-4d96-a786-349c2ed2c04a') { // only let one login at a time for this user!
						withCredentials([usernamePassword(credentialsId: '895d8db1-87c2-4d96-a786-349c2ed2c04a', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
							sh 'appc login --username "$USER" --password "$PASS"'
						}
						sh 'appc ti build -p ios --build-only'
						sh 'appc logout'
					}
					// TODO Test module in app! See https://raw.githubusercontent.com/sgtcoolguy/ci/v8/travis/script.sh
					archiveArtifacts '*.zip'
				} // dir
			} // nodeJs
		} // node
	}
}

timestamps {
	def branches = [failFast: true]
	node {
		stage('Checkout') {
			checkout scm
			stash 'sources'
			// Determine if we need to run android/ios branches!
			if (fileExists('android')) {
				branches['android'] = buildAndroid(nodeVersion, tiSDKVersion, androidAPILevel)
			}
			if (fileExists('ios')) { // TODO Check for 'iphone' folder
				branches['iOS'] = buildIOS(nodeVersion, tiSDKVersion)
			}
		}
	}

	stage('Build') {
		parallel(branches)
	}
} // timestamps
