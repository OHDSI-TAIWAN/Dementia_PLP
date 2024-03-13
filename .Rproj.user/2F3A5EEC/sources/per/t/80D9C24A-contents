# install.packages('remotes')
# remotes::install_github('ohdsi/ROhdsiWebApi')
# remotes::install_github('ohdsi/CohortGenerator')
# remotes::install_github('ohdsi/FeatureExtraction')
# remotes::install_github('ohdsi/PatientLevelPrediction')
# remotes::install_github('ohdsi/DeepPatientLevelPrediction')
# Get the current file path
# options(reticulate.conda_binary = "~/mambaforge/bin/conda")
current_script <- rstudioapi::getSourceEditorContext()$path
setwd(dirname(current_script))

#PatientLevelPrediction::setPythonEnvironment(envname = 'test', envtype='python') 
reticulate::use_virtualenv('test')
# reticulate::use_condaenv('base')

# define target-outcome pair
pairs <- list(
  dm2d = list(
    analysisId = "Type 2 DM to Dementia",
    analysisDesc = "DM2D prediction",
    outcomeId = 243, # outcome cohort ID
    targetId = 242, # population cohort ID
    cohortTable = "dm2dCohort"
  ),
  ht2d = list(
    analysisId = "Hyypertension to Dementia",
    analysisDesc = "HT2D prediction",
    outcomeId = 243, # outcome cohort ID
    targetId = 262, # population cohort ID
    cohortTable = "ht2dCohort"
  ),
  dep2d = list(
    analysisId = "Depression to Dementia",
    analysisDesc = "DEP2D prediction",
    outcomeId = 243, # outcome cohort ID
    targetId = 267, # population cohort ID
    cohortTable = "dep2dCohort"
  )
)

saveDir <- "./out_"
#DatabaseConnector::downloadJdbcDrivers('sql server','~/.config/jdbc/')
conn <- DatabaseConnector::createConnectionDetails(
  'sql server',
  user = 'sa',
  password = 'hiking@tmu2',
  server = '10.164.1.154',
  pathToDriver = '~/.config/jdbc/'
)

for (key in names(pairs)) {
  targetId <- pairs[[key]][['targetId']]
  outcomeId <- pairs[[key]][['outcomeId']]
  cohortTable <- pairs[[key]][['cohortTable']]
  outcomeTable <- pairs[[key]][['cohortTable']]
  
  cohortIds <- c(targetId, outcomeId)  
  
  #ROhdsiWebApi::authorizeWebApi('http://192.168.193.194:8080/WebAPI', authMethod = 'db', webApiUsername = 'admin', webApiPassword = 'admin')
  cohortDef <- ROhdsiWebApi::exportCohortDefinitionSet(
    baseUrl = 'http://10.164.1.154:8080/WebAPI', 
    cohortIds = cohortIds
  )
  databaseDetails <- PatientLevelPrediction::createDatabaseDetails(
    connectionDetails = conn,
    cdmDatabaseSchema = 'OHDSI_V5_V2.dbo',
    cdmDatabaseName = 'TMU CRD',
    cdmDatabaseId = 'tmudb',
    cohortDatabaseSchema = 'OHDSI_ACHILLES.dbo',
    outcomeDatabaseSchema = 'OHDSI_ACHILLES.dbo',
    cohortTable = cohortTable,
    outcomeTable = cohortTable,
    targetId = targetId,
    outcomeIds = outcomeId,
    cdmVersion = 5
  )
  tableNames<-CohortGenerator::getCohortTableNames(cohortTable = cohortTable)
  CohortGenerator::createCohortTables(
    connectionDetails = conn,
    cohortDatabaseSchema = 'OHDSI_ACHILLES.dbo',
    cohortTableNames = tableNames
  )
  cohortGen<-CohortGenerator::generateCohortSet(
    conn,
    cdmDatabaseSchema = 'OHDSI_V5_V2.dbo',
    tempEmulationSchema = NULL,
    cohortDatabaseSchema = 'OHDSI_ACHILLES.dbo',
    cohortTableNames = tableNames,
    cohortDefinitionSet = cohortDef
  )
  covariateSettings <- FeatureExtraction::createCovariateSettings(
    useDemographicsGender = TRUE,
    useDemographicsAgeGroup = TRUE,
    useDemographicsIndexMonth = TRUE,
    useDemographicsPriorObservationTime = TRUE,
    useDemographicsPostObservationTime = TRUE,
    useDemographicsTimeInCohort = TRUE,
    useConditionOccurrenceAnyTimePrior = TRUE,
    useConditionOccurrenceLongTerm = TRUE,
    useConditionOccurrenceMediumTerm = TRUE,
    useConditionOccurrenceShortTerm = TRUE,
    useConditionEraAnyTimePrior = TRUE,
    useConditionGroupEraLongTerm = TRUE,
    useConditionGroupEraShortTerm = F,
    useDrugGroupEraLongTerm = TRUE,
    useDrugGroupEraShortTerm = F,
    useDrugGroupEraOverlapping = TRUE,
    useDrugExposureAnyTimePrior = TRUE,
    useDrugExposureLongTerm = TRUE,
    useDrugExposureMediumTerm = TRUE,
    useDrugExposureShortTerm = TRUE,
    useDcsi = TRUE,
    useChads2 = TRUE,
    useChads2Vasc = TRUE,
    useCharlsonIndex = TRUE  
  )
  restrictPlpDataSettings <- PatientLevelPrediction::createRestrictPlpDataSettings(sampleSize = NULL)
  populationSettings <- PatientLevelPrediction::createStudyPopulationSettings(
    washoutPeriod = 365,
    firstExposureOnly = F,
    removeSubjectsWithPriorOutcome = T,
    priorOutcomeLookback = 365,
    riskWindowStart = 1,
    riskWindowEnd = (1825),
    startAnchor = 'cohort start',
    endAnchor = 'cohort start',
    minTimeAtRisk = 1,
    requireTimeAtRisk = T,
    includeAllOutcomes = T
  )
  splitSettings <- PatientLevelPrediction::createDefaultSplitSetting(
    trainFraction = 0.8,
    testFraction = 0.2,
    splitSeed = 1
  )
  sampleSettings <- PatientLevelPrediction::createSampleSettings()
  featureEngineeringSettings <- PatientLevelPrediction::createFeatureEngineeringSettings()
  preprocessSettings <- PatientLevelPrediction::createPreprocessSettings(
    minFraction = 0.01,
    normalize = T,
    removeRedundancy = T
  )
  models <- list(
    PatientLevelPrediction::setLassoLogisticRegression(),
    PatientLevelPrediction::setLightGBM(
      isUnbalance = T,
      seed=1,
      numLeaves = c(20),
      minDataInLeaf = c(10)
    ),
    # PatientLevelPrediction::setAdaBoost(),
    # PatientLevelPrediction::setRandomForest()
    # DeepPatientLevelPrediction::setMultiLayerPerceptron()
    # PatientLevelPrediction::setMLP(),
    PatientLevelPrediction::setKNN()
  )
  modelList <- list()
  
  for (i in 1:length(models)){
    modelList[[i]] <- PatientLevelPrediction::createModelDesign(
      targetId = targetId,
      outcomeId = outcomeId,
      restrictPlpDataSettings = restrictPlpDataSettings,
      populationSettings = populationSettings,
      featureEngineeringSettings = featureEngineeringSettings,
      sampleSettings = sampleSettings,
      preprocessSettings = preprocessSettings,
      modelSettings = models[[i]],
      splitSettings = splitSettings,
      runCovariateSummary = T
    )
  }
  
  PatientLevelPrediction::runMultiplePlp(
    databaseDetails = databaseDetails,
    modelDesignList = modelList,
    logSettings = PatientLevelPrediction::createLogSettings(
      verbosity = "DEBUG", timeStamp = T, logName = "runPlp Log"),
    saveDirectory = paste0(saveDir, key),
    sqliteLocation = file.path(paste0(saveDir,key), 'sqlite')
  )
}

PatientLevelPrediction::viewMultiplePlp('out_dm2d')
PatientLevelPrediction::viewMultiplePlp('out_ht2d')
PatientLevelPrediction::viewMultiplePlp('out_dep2d')

zip('result.zip', files = c("out_dep2d", "out_dm2d", "out_ht2d"))
# plpData <- PatientLevelPrediction::getPlpData(
#   databaseDetails = databaseDetails,
#   covariateSettings = covariateSettings,
#   restrictPlpDataSettings = restrictPlpDataSettings
# )
# # 
# PatientLevelPrediction::savePlpData(plpData, dataOut)
# 
# plpData <- PatientLevelPrediction::loadPlpData(dataOut)


# install anaconda first

# PatientLevelPrediction::setLassoLogisticRegression(
#   variance = 0.01,
#   seed = 1,
#   includeCovariateIds = c(),
#   noShrinkage = c(0),
#   threads = -1,
#   forceIntercept = F,
#   upperLimit = 30,
#   lowerLimit = 0.01,
#   tolerance = 2e-06,
#   maxIterations = 3000,
#   priorCoefs = NULL    
# )

# library(DeepPatientLevelPrediction)






# table 1
# settings <- FeatureExtraction::createTable1CovariateSettings()
# 
# covStroke <- FeatureExtraction::getDbCovariateData(connectionDetails = conn,
#                                                    cdmDatabaseSchema = 'OHDSI_V5.dbo',
#                                                    cohortDatabaseSchema = 'OHDSI_ACHILLES.dbo',
#                                                    cohortTable = "psciCohort",
#                                                    cohortId = targetId,
#                                                    covariateSettings = settings,
#                                                    aggregated = TRUE)
# 
# covCI <- FeatureExtraction::getDbCovariateData(connectionDetails = conn,
#                                                cdmDatabaseSchema = 'OHDSI_V5.dbo',
#                                                cohortDatabaseSchema = 'OHDSI_ACHILLES.dbo',
#                                                cohortTable = "psciCohort",
#                                                cohortId = outcomeId,
#                                                covariateSettings = settings,
#                                                aggregated = TRUE)
# 
# std <- FeatureExtraction::computeStandardizedDifference(covStroke, covCI)
# result <- FeatureExtraction::createTable1(covStroke, covCI)
# print(result, row.names = T, right = F)
# write.csv(result, file = "tableone.csv", row.names = FALSE)
