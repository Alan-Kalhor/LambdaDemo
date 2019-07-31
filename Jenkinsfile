
node {

	
	def DOTNET_PATH = '/home/ec2-user/dotnet'
	def FUNCTION_NAME = 'lambdademo'
	def APP_MAIN_FOLDER = 'lambdademo'
	def S3_BUCKET = "s3.alankalhor.${FUNCTION_NAME}"
	def REGION = 'ap-southeast-2'
	def PROD_ALIAS = 'production'
	def STAGING_ALIAS = 'staging'

	
	def lambdaVersion = ''
	
    stage('Checkout'){
        checkout scm
		sh "echo 'after git checkout ...'"		
		sh "ls"
    }
	
	stage('Build') {
/*		sh "sudo $DOTNET_PATH/dotnet clean ${FUNCTION_NAME}"	
		sh "sudo $DOTNET_PATH/dotnet build ${FUNCTION_NAME} --configuration Release"
*/
		sh "sudo dotnet restore"	
		sh "sudo dotnet clean"	
		sh "sudo dotnet build --configuration Release"		
	}
	
	stage('Test') {
		sh "sudo dotnet test"
	}
	

	stage('Deploy') {
	
		env.DOTNET_ROOT = "/usr/share/dotnet"
		env.PATH = "$PATH:/usr/share/dotnet"

		sh "sudo dotnet-lambda list-functions"
		
		sh "echo 'about to create s3'"		
		try {
			sh "aws s3api create-bucket --bucket ${S3_BUCKET} --create-bucket-configuration  LocationConstraint=${REGION}"
		}
		catch (exc) {
			sh "echo 's3 bucket already exists'"		
		}
	
		sh "echo 'about to deploy lambda'"		
		dir("${APP_MAIN_FOLDER}") {
			//sh "$DOTNET_PATH/dotnet-lambda deploy-function DotNetCoreWithTest1 --function-role JenkinsBuildRole"
			sh "$dotnet-lambda deploy-function --function-runtime dotnetcore2.1 --function-name ${FUNCTION_NAME}  --function-memory-size 256 --function-timeout 30 --function-role mydotnetroll --function-handler ${FUNCTION_NAME}::${FUNCTION_NAME}.LambdaEntryPoint::FunctionHandlerAsync --disable-interactive true"
			
			sh "echo 'after deploy-function'"						
			
			sh "$DOTNET_PATH/dotnet-lambda deploy-serverless ${FUNCTION_NAME} --s3-bucket ${S3_BUCKET} --stack-name ${FUNCTION_NAME}"

		}
	}

	
	if (env.BRANCH_NAME == 'master') {
		stage('Publish') {
		
			lambdaVersion = sh(
				script: "aws lambda publish-version --function-name ${FUNCTION_NAME} --region ${REGION} | jq -r '.Version'",
				returnStdout: true
			)
			sh "echo $lambdaVersion"						
		}


		stage('Deploy to Staging?') {
			milestone 2
			input "Do you want to deploy to Staging?"
		}

		stage('Deploy to Staging') {
			def check_staging_alias = sh(
				script: "aws lambda list-aliases --function-name ${FUNCTION_NAME} --region ${REGION} | jq -r '.Aliases[] | select(.Name == \"${STAGING_ALIAS}\") | 1'",
				returnStdout: true
			)
			
			if (!check_staging_alias.trim().equals("1")) {
				sh "aws lambda create-alias --function-name ${FUNCTION_NAME} --name ${STAGING_ALIAS} --region ${REGION} --function-version ${lambdaVersion}"
			}
			else {
				sh "aws lambda update-alias --function-name ${FUNCTION_NAME} --name ${STAGING_ALIAS} --region ${REGION} --function-version ${lambdaVersion}"			
			}
		}
		
		stage('Deploy to Production?') {
			milestone 3
			input "Do you want to deploy to Production?"
		}		
		
		stage('Deploy to Production') {
			def check_prod_alias = sh(
				script: "aws lambda list-aliases --function-name ${FUNCTION_NAME} --region ${REGION} | jq -r '.Aliases[] | select(.Name == \"${PROD_ALIAS}\") | 1'",
				returnStdout: true
			)
		
			if (!check_prod_alias.trim().equals("1")) {
				sh "aws lambda create-alias --function-name ${FUNCTION_NAME} --name ${PROD_ALIAS} --region ${REGION} --function-version ${lambdaVersion}"
			}
			else {
				sh "aws lambda update-alias --function-name ${FUNCTION_NAME} --name ${PROD_ALIAS} --region ${REGION} --function-version ${lambdaVersion}"
			}		
		}
	}	
	
}
