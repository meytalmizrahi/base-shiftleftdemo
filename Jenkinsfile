node {
    files= ['deploy.yml']

    withCredentials([usernamePassword(credentialsId: 'prisma_cloud', passwordVariable: 'PC_PASS', usernameVariable: 'PC_USER')]) {
    PC_TOKEN = sh(script:"curl -s -k -H 'Content-Type: application/json' -H 'accept: application/json' --data '{\"username\":\"$PC_USER\", \"password\":\"$PC_PASS\"}' https://${AppStack}/login | jq --raw-output .token", returnStdout:true).trim()
    }

    stage('Clone repository') {
        checkout scm
    }


    stage('Check Git dependencies has no vulnerabilities') {
        try {
            withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
                sh('chmod +x files/checkGit.sh && ./files/checkGit.sh')
            }
        } catch (err) {
            echo err.getMessage()
            echo "Error detected"
			throw RuntimeException("Build failed for some specific reason!")
        }
    }

    stage('Scan IaC with BC/Checkov') {
		try {
	     		withCredentials([
            				string(
              				credentialsId: 'bc-api-key',
              				variable: 'BC_API')
             			]) {
			response = sh(script:"checkov --file files/deploy.yml --bc-api-key $BC_API --repo-id meytalmizrahi/base-shiftleftdemo -b main -o junitxml > result.xml || true", returnStdout:true).trim() // -o junitxml > result.xml || true"
            			}
		
	    		response = sh(script:"cat result.xml", returnStdout:true)
	     		print "${response}"
             			junit skipPublishingChecks: true, testResults: "result.xml"
		                }	
		catch (err) {
            			echo err.getMessage()
            			echo "Error detected"
		            }
    }

    //$PC_USER,$PC_PASS,$PC_CONSOLE when Galileo is released. 
    stage('Apply security policies (Policy-as-Code) for evilpetclinic') {
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh('chmod +x files/addPolicies.sh && ./files/addPolicies.sh')
        }
    }

    //$PC_USER,$PC_PASS,$PC_CONSOLE when Galileo is released. 
    stage('Download latest twistcli') {
        withCredentials([usernamePassword(credentialsId: 'prisma_cloud', passwordVariable: 'PC_PASS', usernameVariable: 'PC_USER')]) {
            sh 'curl -k -u $PC_USER:$PC_PASS --output ./twistcli https://$PC_CONSOLE/api/v1/util/twistcli'
            sh 'sudo chmod a+x ./twistcli'
        }
    }

    stage('Scan image with twistcli') {
        try {
	    sh 'docker pull solalraveh/evilpetclinic:latest'
        withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
            sh 'curl -k -u $TL_USER:$TL_PASS --output ./twistcli https://$TL_CONSOLE/api/v1/util/twistcli'
            sh 'sudo chmod a+x ./twistcli'
            sh "./twistcli images scan --u $TL_USER --p $TL_PASS --address https://$TL_CONSOLE --details solalraveh/evilpetclinic:latest"
            }
        } 
        catch (err) {
            echo err.getMessage()
            echo "Error detected"
			throw RuntimeException("Build failed for some specific reason!")
        }
    }

    stage ('Prisma Cloud Scan using Jenkins Plugin') {
        	// Scan the image
            prismaCloudScanImage ca: '',
            cert: '',
            dockerAddress: 'unix:///var/run/docker.sock',
            image: 'solalraveh/evilpetclinic:latest',
            key: '',
            logLevel: 'info',
            podmanPath: '',
            project: '',
            resultsFile: 'prisma-cloud-scan-results.json',
            ignoreImageBuildTime:true
    }
    
    stage ('Publish Prisma Cloud Results') {
            // The post section lets you run the publish step regardless of the scan results
	    echo 'Publish Prisma Cloud Results'
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
       
    }
// stage ("Scan K8S Yaml Manifest with BC/Checkov") {
    //   withDockerContainer (image: 'bridgecrew/jenkins_bridgecrew_runner:latest') {
    //    sh "/run.sh cadc031b-f0a7-5fe1-9085-e0801fc52131 https://github.com/meytalmizrahi/base-shiftleftdemo"
    //}
//}

    stage("Scan Cloud Formation Template with API v2") {

        def response

        response = sh(script:"curl -sq -X POST -H 'x-redlock-auth: ${PC_TOKEN}' -H 'Content-Type: application/vnd.api+json' --url https://${AppStack}/iac/v2/scans --data-binary '@scan-asset.json'", returnStdout:true).trim()

        def SCAN_ASSET = readJSON text: response

        def SCAN_ID = SCAN_ASSET['data'].id
        def SCAN_URL = SCAN_ASSET['data']['links'].url

        //Upload files
        sh(script:"curl -sq -X PUT  --url '${SCAN_URL}' -T 'files/deploy.yml'", returnStdout:true).trim()

        //start the Scan
        response = sh(script:"curl -sq -X POST -H 'x-redlock-auth: ${PC_TOKEN}' -H 'Content-Type: application/vnd.api+json' --url https://${AppStack}/iac/v2/scans/${SCAN_ID} --data-binary '@scan-start-k8s.json'", returnStdout:true).trim()


        //Get the Status
        def SCAN_STATUS
        def STATUS

        //Need a Do-While loop here.   Haven't found a good syntax with Groovy in Jenkins
        response = sh(script:"curl -sq -H 'x-redlock-auth: ${PC_TOKEN}' -H 'Content-Type: application/vnd.api+json' --url https://${AppStack}/iac/v2/scans/${SCAN_ID}/status", returnStdout:true).trim()
        SCAN_STATUS = readJSON text: response
        STATUS = SCAN_STATUS['data']['attributes']['status']

        while  (STATUS == 'processing'){
            response = sh(script:"curl -sq -H 'x-redlock-auth: ${PC_TOKEN}' -H 'Content-Type: application/vnd.api+json' --url https://${AppStack}/iac/v2/scans/${SCAN_ID}/status", returnStdout:true).trim()
            SCAN_STATUS = readJSON text: response
            STATUS = SCAN_STATUS['data']['attributes']['status']
            print "${STATUS}"

        }


        //Get the Results
        response = sh(script:"curl -sq -H 'x-redlock-auth: ${PC_TOKEN}' -H 'Content-Type: application/vnd.api+json' --url https://${AppStack}/iac/v2/scans/${SCAN_ID}/results", returnStdout:true).trim()
        def SCAN_RESULTS = readJSON text: response

        print "${SCAN_RESULTS}"

    }


//   files.each { item ->
//        stage("Scan IaC file ${item} with twistcli") {
//            try {
//                withCredentials([usernamePassword(credentialsId: 'prisma_cloud', passwordVariable: 'PC_PASS', usernameVariable: 'PC_USER')]) {
//                    //sh "./twistcli iac scan --u $PC_USER --p $PC_PASS --asset-name "Jenkins IaC" --tags env:jenkins --compliance-threshold high --address https://$PC_CONSOLE --files files/${item}"
//                    sh "./twistcli iac scan --u $PC_USER --p $PC_PASS --type k8s --asset-name evilpetclinic --compliance-threshold medium --address https://$PC_CONSOLE files/${item}"
//                }
//            } catch (err) {
//                echo err.getMessage()
//                echo "Error detected"
//                throw RuntimeException("Build failed for some specific reason!")
//            }
//        }
//    }

    stage('Deploy evilpetclinic') {
        sh 'kubectl create ns evil --dry-run -o yaml | kubectl apply -f -'
        sh 'kubectl delete --ignore-not-found=true -f files/deploy.yml -n evil'
        sh 'kubectl apply -f files/deploy.yml -n evil'
        sh 'sleep 10'
    }

    stage('Run bad Runtime attacks') {
        sh('chmod +x files/runtime_attacks.sh && ./files/runtime_attacks.sh')
    }

    stage('Run bad HTTP stuff for WAAS to catch') {
        sh('chmod +x files/waas_attacks.sh && ./files/waas_attacks.sh')
    }
	

}