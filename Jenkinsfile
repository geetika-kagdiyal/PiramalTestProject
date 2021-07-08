#!groovy

node 
{

    def SF_CONSUMER_KEY=env.SF_CONSUMER_KEY
    def SF_USERNAME=env.SF_USERNAME
    def SERVER_KEY_CREDENTIALS_ID=env.SERVER_KEY_CREDENTIALS_ID
    def SF_INSTANCE_URL = env.SF_INSTANCE_URL ?: "https://login.salesforce.com"
	def TEST_LEVEL='%Testlevel%'
	def DELTA='DeltaChanges'
	def DEPLOYDIR='toDeploy'
	def APIVERSION='51.0'
    def toolbelt = tool 'toolbelt'
	def scannerHome = tool 'SonarScanner'

	//----------------------------------------------------------------------
	//Check if Previous and Latest commit IDs are provided.
	//----------------------------------------------------------------------
	
	if ((params.PreviousCommitId == '') || (params.LatestCommitId == ''))
	{
		error("Please enter Previous and Latest commit IDs both")
	}
	
	//----------------------------------------------------------------------
	//Check if Test classes are mentioned in case of RunSpecifiedTests.
	//----------------------------------------------------------------------
		
	if (TESTLEVEL=='RunSpecifiedTests')
	{
		if (params.SpecifyTestClass == '')
		{
			error("Please Specify Test classes")
		}
	}


    stage('Clean Workspace') 
	{
        try 
		{
            deleteDir()
			currentBuild.displayName = "Crowley Demo Pipeline ${BUILD_NUMBER}"
        }
        catch (Exception e) 
		{
            println('Unable to Clean WorkSpace.')
        }
    }
    // -------------------------------------------------------------------------
    // Check out code from source control.
    // -------------------------------------------------------------------------

    stage('checkout source') 
	{
        checkout scm
		def branchOriginName = bat (label: 'Branch name', script: '@git name-rev --name-only HEAD', returnStdout: true).trim() as String   
        branchName = branchOriginName.replaceAll('remotes/origin/','').split('~')[0]
        println branchName
    
    }


    // -------------------------------------------------------------------------
    // Run all the enclosed stages with access to the Salesforce
    // JWT key credentials.
    // -------------------------------------------------------------------------

 	withEnv(["HOME=${env.WORKSPACE}"]) 
	{	
	
	    withCredentials([file(credentialsId: SERVER_KEY_CREDENTIALS_ID, variable: 'server_key_file')]) 
		{
			// -------------------------------------------------------------------------
			// Authenticate to Salesforce using the server key.
			// -------------------------------------------------------------------------

			stage('Authorize to Salesforce') 
			{
				rc = command "${toolbelt}/sfdx auth:jwt:grant --instanceurl ${SF_INSTANCE_URL} --clientid ${SF_CONSUMER_KEY} --jwtkeyfile ${server_key_file} --username ${SF_USERNAME} --setalias SFDX"
				if (rc != 0) 
				{
					error 'Salesforce org authorization failed.'
				}
			}


			// -------------------------------------------------------------------------
			// Generate delta changes
			// -------------------------------------------------------------------------
			stage('Delta changes')
			{
                script
                {
					rc = command "${toolbelt}/sfdx sfpowerkit:project:diff --revisionfrom %PreviousCommitId% --revisionto %LatestCommitId% --output ${DELTA} --apiversion ${APIVERSION} -x"
						
					def folder = fileExists 'DeltaChanges/force-app'
					def file = fileExists 'DeltaChanges/destructiveChanges.xml'
    
					if( folder && !file )
					{
						dir("${WORKSPACE}/${DELTA}")
						{
							println "Force-app folder exist, destructiveChanges.xml doesn't exist"
							rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
						}
					} 
					else if ( !folder && file ) 
					{
						bat "copy destructive\\package.xml ${DELTA}"
						println "Force-app folder doesn't exist, destructiveChanges.xml exist" 
					}
					else if ( folder && file ) 
					{
						dir("${WORKSPACE}/${DELTA}")
						{
							println "Force-app folder exist, destructiveChanges.xml exist"
							if (Deployment_Type=='Deploy Only')
							{
								println "You have selected deploy only, deleting destructivechanges.xml to avoid component deletion."
								bat "del /f destructiveChanges.xml"
								rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
							}
							else if (Deployment_Type=='Delete and Deploy')
							{
								println "You have selected Delete and Deploy, both deletion and deployment will be performed."
								rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
								bat "copy destructiveChanges.xml ..\\${DEPLOYDIR}"
							}
							else if (Deployment_Type=='Validate Only')
							{
								println "You have selected Validate Only, so only validation will be performed."
								rc = command "${toolbelt}/sfdx force:source:convert -d ../${DEPLOYDIR}"
								//bat "copy destructiveChanges.xml ..\\${DEPLOYDIR}"
							}
							else if (Deployment_Type=='Delete Only')
							{
								println "You have selected Delete only but force-app folder also exists. Deleting the force-app folder to avoid deploying a component."
								bat "echo y | rmdir /s force-app"
								bat "copy ..\\destructive\\package.xml ."

							}
						}
					}

					else 
					{
						println "No Delta changes found."
					}
		
                }
			}
			
			// -------------------------------------------------------------------------
			// Analysis of delta changes on SonarCloud
			// -------------------------------------------------------------------------
			
			stage('SonarCloud Analysis') 
			{
				if (SonarCloud_Analysis=='true')
				script
				{
					println "You have opted for SonarCloud Analysis."
					def folder = fileExists 'toDeploy'
					if( folder )
					{
						dir("${WORKSPACE}")
						{
							withSonarQubeEnv(credentialsId: '7a7722f5-c99a-4fdb-a023-1083edb5dc06') 
							{
								println "Analysing delta changes on SonarCloud"
								bat "${scannerHome}/sonar-scanner.bat -Dproject.settings=./sonar-scanner.properties"
							}
						}
					}
					else
					{
						println "No toDeploy folder to scan, so skip this step."
					}
				}
				else if (SonarCloud_Analysis=='false')
				{
					println "You have not opted for SonarCloud Analysis."
				}
    			
			} 
					
			// -------------------------------------------------------------------------
			// Example shows how to run a check-only deploy.
			// -------------------------------------------------------------------------
			
			stage('Validate Only') 
			{
				if (Deployment_Type=='Validate Only')
				{
				
					script
					{
					
						if (TESTLEVEL=='NoTestRun') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} "
						}
						else if (TESTLEVEL=='RunLocalTests') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} --verbose --loglevel fatal"
						}
						else if (TESTLEVEL=='RunSpecifiedTests')
						{
							println TESTLEVEL
							def Testclass = SpecifyTestClass.replaceAll('\\s','')
							println Testclass
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --checkonly --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} -r ${Testclass} --verbose --loglevel fatal"
						}
	
						else (rc != 0) 
						{
							error 'Validation failed.'
						}
					}
				}
				else
				{
					return
				}
			}
			
			// -------------------------------------------------------------------------
			// Deploy metadata and execute unit tests.
			// -------------------------------------------------------------------------			

			stage('Deploy and Run Tests') 
			{
				if (Deployment_Type=='Deploy Only')
				{	

					script
					{
						if (TESTLEVEL=='NoTestRun') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} "
						}
						else if (TESTLEVEL=='RunLocalTests') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} --verbose --loglevel fatal"
						}
						else if (TESTLEVEL=='RunSpecifiedTests') 
						{
							println TESTLEVEL
							def Testclass = SpecifyTestClass.replaceAll('\\s','')
							println Testclass
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} -r ${Testclass} --verbose --loglevel fatal"
						}
						else (rc != 0) 
						{
							error 'Salesforce deployment failed at deploy only step.'
						}
					}
				}
				else
				{
					return
				}
			}
		
			// -------------------------------------------------------------------------
			// Delete metadata.
			// -------------------------------------------------------------------------

			stage('Delete Components') 
			{
				if (Deployment_Type=='Delete Only')
				{	
					rc = command "${toolbelt}/sfdx force:mdapi:deploy --targetusername ${SF_USERNAME} -d ${DELTA} --wait 10"
				}
				else
				{
					return
				}
			}

			// -------------------------------------------------------------------------
			// Delete and Deploy metadata and execute unit tests.
			// -------------------------------------------------------------------------			

			stage('Delete, Deploy and Run Tests') 
			{
				if (Deployment_Type=='Delete and Deploy')
				{	

					script
					{
						if (TESTLEVEL=='NoTestRun') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} "
						}
						else if (TESTLEVEL=='RunLocalTests') 
						{
							println TESTLEVEL
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} --verbose --loglevel fatal"
						}
						else if (TESTLEVEL=='RunSpecifiedTests') 
						{
							println TESTLEVEL
							def Testclass = SpecifyTestClass.replaceAll('\\s','')
							println Testclass
							rc = command "${toolbelt}/sfdx force:mdapi:deploy -d ${DEPLOYDIR} --wait 10 --targetusername ${SF_USERNAME} --testlevel ${TESTLEVEL} -r ${Testclass} --verbose --loglevel fatal"
						}
						else (rc != 0) 
						{
							error 'Salesforce deployment failed at Delete and Deploy step.'
						}
					}
				}
				else
				{
					return
				}
			}

		}		    
	  
	}
}

def command(script) 
{
    if (isUnix()) 
	{
        return sh(returnStatus: true, script: script);
    } 
	else 
	{
		return bat(returnStatus: true, script: script);
    }
}
