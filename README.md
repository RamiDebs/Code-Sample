// InitializeRoutes sets up the routes for the API by attaching the route handler functions to the router.
func InitializeRoutes(router *gin.Engine) {
    router.Use(database.Connect) // Use the custom database connection middleware
    // Initialize routes for banned words
    bannedWordsApi.InitBannedWordsRoutes(router)
    admin.InitUploadDataRoutes(router)
}

func InitializeInterceptors(router *gin.Engine) {
    router.Use(middleware.Logger())
    router.Use(middleware.CORSMiddleware())
}


func InitBannedWordsRoutes(router *gin.Engine) {
    // Group routes under /api
    apiRoutes := router.Group("/api")

    // Group routes under /api/bannedWords
    bannedWordsRoutes := apiRoutes.Group("/bannedWords")

    // Define the /language-data/:langID route under /api/bannedWords
    bannedWordsRoutes.GET("/language-data/:langID", GetLanguageData)
    bannedWordsRoutes.POST("/languages-data", GetLanguagesData)
    bannedWordsRoutes.POST("/suggest-word", SetSuggestedWord)
}

func GetLanguagesData(c *gin.Context) {
    var langIDs []int
    if err := c.BindJSON(&langIDs); err != nil {
        response.SendErrorResponse(c, http.StatusBadRequest, "Invalid request data", err)
        return
    }

    var languagesData []model.LanguageResponse
    for _, langID := range langIDs {
        var langData model.LanguageData
        var bannedWords []model.BannedWords
        var alternativeChars []model.AlternativeCharacters

        // Assuming db is your database connection
        if err := database.DB.Where("id = ?", langID).First(&langData).Error; err != nil {
            continue // Skip if langID is not found
        }

        database.DB.Where("lang_id = ? AND force_apply != ?", langID, true).Find(&bannedWords)
        database.DB.Where("lang_id = ?", langID).Find(&alternativeChars)

        languagesData = append(languagesData, model.LanguageResponse{
            LangCode:         langData.LangCode,
            ID:               int(langData.ID),
            BannedWords:      bannedWords,
            AlternativeChars: alternativeChars,
        })
    }

    response.SendSuccessResponse(c, http.StatusOK, "Data retrieved successfully", languagesData)
}


