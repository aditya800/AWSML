# An Idiot's Guide to Implementing AWS's ML Services in iOS: Sample

This is a comprehensive, easy to understand, dumbed-down, one stop library, guide, or cheat sheet to leverage the AWS ML suite of applications in your iOS app for hackathons or personal projects. Since Amazon Personalise and Amazon Forecast are not yet released, this guide does not include them. 

(This is only a sample. Please purchase the document for access to all functions).  

Despite its powerful performance and market share (more than 50% in Q32018), AWS ML is still seen as something like a black box. 

This guide not only provides a direct library of routines for various AWS services but also expands the functions of those ML services that cannot be directly implemented in a native environment such as AWS Comprehend (NLP). 

It is assumed that you have created an AWS account and also created the cognito identity pool and CLI to configure AWS for your iOS application    

1. Input Image and get back a list of objects/items/things in that picture / Use AWS for image recognition in Swift 

This is fairly simple, once you have stored the image as a variable, you can use AWS Rekognition to convert it into a list:

	func convertToList(image: UIImage) {
        
        let sourceImage = image
        let rekognitionClient = AWSRekognition.default()
        
        let image = AWSRekognitionImage()
        image!.bytes = UIImageJPEGRepresentation(sourceImage, 0.7)
        
        guard let request = AWSRekognitionDetectLabelsRequest() else {
            print("Unable to initialize AWSRekognitionDetectLabelsRequest.")
            return }
        
        var str = " "
        
        request.image = image
        request.maxLabels = 20
        request.minConfidence = 90
        
        rekognitionClient.detectLabels(request) {
            (result, error) in
            if error != nil {
                print(error!)
                return
            }
            
            if result != nil {
                
                print("\n\n")
            // A count variable can be used here to keep track of the number of objects 
                var strarray = [String]()
                
                for (index, val) in result!.labels!.enumerated() {
                    
                    //print(val.name!)
                    strarray.append(val.name!)
                    if index == 0 {
                        
                        str = str + " " + val.name!
                    } else {
                        str = str + " " + val.name!
                    }
                }
                
                print(strarray)
 
                   } else {
                print("No result")
            }
            
       }
        
    }


2. Detect celebrity faces in a given image / Use AWS for recognising celebrities in Swift 

This is done by using the AWS Celebrities Request in the rekognition api. 

	func sendImageToRekognition(celebImageData: Data){
        
        //Delete older labels or buttons
        DispatchQueue.main.async {
            [weak self] in
            for subView in (self?.CelebImageView.subviews)! {
                subView.removeFromSuperview()
            }
        }
        
        rekognitionObject = AWSRekognition.default()
        let celebImageAWS = AWSRekognitionImage()
        celebImageAWS?.bytes = celebImageData
        let celebRequest = AWSRekognitionRecognizeCelebritiesRequest()
        celebRequest?.image = celebImageAWS
        
        rekognitionObject?.recognizeCelebrities(celebRequest!){
            (result, error) in
            if error != nil{
                print(error!)
                return
            }
            
            //1. First we check if there are any celebrities in the response
            if ((result!.celebrityFaces?.count)! > 0){
                
                //2. Celebrities were found. Lets iterate through all of them
                for (index, celebFace) in result!.celebrityFaces!.enumerated(){
                    
                    //Check the confidence value returned by the API for each celebirty identified
                    if(celebFace.matchConfidence!.intValue > 50){ //Adjust the confidence value to whatever you are comfortable with
                        
                        //We are confident this is celebrity. Lets point them out in the image using the main thread
                        DispatchQueue.main.async {
                            [weak self] in
                            
                            //Create an instance of Celebrity. This class is availabe with the starter application you downloaded
                            let celebrityInImage = Celebrity()
                            
                            celebrityInImage.scene = (self?.CelebImageView)!
                            
                            //Get the coordinates for where this celebrity face is in the image and pass them to the Celebrity instance
                            celebrityInImage.boundingBox = ["height":celebFace.face?.boundingBox?.height, "left":celebFace.face?.boundingBox?.left, "top":celebFace.face?.boundingBox?.top, "width":celebFace.face?.boundingBox?.width] as! [String : CGFloat]
                            
                            //Get the celebrity name and pass it along
                            celebrityInImage.name = celebFace.name!
                            //Get the first url returned by the API for this celebrity. This is going to be an IMDb profile link
                            if (celebFace.urls!.count > 0){
                                celebrityInImage.infoLink = celebFace.urls![0]
                            }
                                //If there are no links direct them to IMDB search page
                            else{
                                celebrityInImage.infoLink = "https://www.imdb.com/search/name-text?bio="+celebrityInImage.name
                            }
                            //Update the celebrity links map that we will use next to create buttons
                            self?.infoLinksMap[index] = "https://"+celebFace.urls![0]
                            
                            //Create a button that will take users to the IMDb link when tapped
                            let infoButton:UIButton = celebrityInImage.createInfoButton()
                            infoButton.tag = index
                            infoButton.addTarget(self, action: #selector(self?.handleTap), for: UIControlEvents.touchUpInside)
                            self?.CelebImageView.addSubview(infoButton)
                        }
                    }
                    
                }
            }
                //If there were no celebrities in the image, lets check if there were any faces (who, granted, could one day become celebrities)
            else if ((result!.unrecognizedFaces?.count)! > 0){
                //Faces are present. Point them out in the Image (left as an exercise for the reader)
                /**/
            }
            else{
                //No faces were found (presumably no people were found either)
                print("No faces in this pic")
            } 
        }
        
    }


3. Converting Text to Speech Using Machine Learning / Use AWS Polly to convert Text to Speech

Mentioning machine learning is important because non ML apis also exist to perform TTS. The use of ML apis, however, for TTS ensures a low latency and high quality translation. In other words, this is more human and natural translation of the text which can be customised as opposed to a robotic one.

Amazon uses Polly to perform conversions

The following implementation example is from the AWS website itself, it is self-explanatory and easy to understand:

	var audioPlayer = AVPlayer()

	func(text: String) {

	let input = AWSPollySynthesizeSpeechURLBuilderRequest()

	// Text to synthesize
	input.text = text 

	// We expect the output in MP3 format
	input.outputFormat = AWSPollyOutputFormat.mp3

	// Choose the voice ID
	input.voiceId = AWSPollyVoiceId.joanna

	// Create an task to synthesize speech using the given synthesis input
	let builder = AWSPollySynthesizeSpeechURLBuilder.default().getPreSignedURL(input)

	// Request the URL for synthesis result
	builder.continueOnSuccessWith(block: { (awsTask: AWSTask<NSURL>) -> Any? in
		// The result of getPresignedURL task is NSURL.
		// Again, we ignore the errors in the example.
		let url = awsTask.result!

		// Try playing the data using the system AVAudioPlayer
		self.audioPlayer.replaceCurrentItem(with: AVPlayerItem(url: url as URL))
		self.audioPlayer.play()

		return nil
	}
}

For this example, the voice of Joanna is used. This voice can be of any other language or gender. To check the list of voices available for use, go here: https://docs.aws.amazon.com/polly/latest/dg/voicelist.html


4. Convert Speech to Text using Machine Learning / Using AWS Transcribe in Swift 

		var audioPlayer = AVPlayer()

		func(text: String) {

		let input = AWSTranscribeRequest()

		input.text = text 

		// We expect the input in MP3 format
		input.outputFormat = Format.mp3

		// Create an task to synthesize speech using the given synthesis input
		let builder = AWSTranscribe.default().getPreSignedURL(input)

		// Request the URL for synthesis result
		builder.continueOnSuccessWith(block: { (awsTask: AWSTask<NSURL>) -> Any? in
			// The result of getPresignedURL task is NSURL.
			// Again, we ignore the errors in the example.
			let url = awsTask.result!

			// Try playing the data using the system AVAudioPlayer
			self.audioPlayer.replaceCurrentItem(with: AVPlayerItem(url: url as URL))
			self.audioPlayer.play()

			return nil
		}
		}


5. Translate words from one language to another / Use AWS Translate in Swift 

Non-iOS Usage/Example: 

1. Copy the following text into a JSON file called translate.json:

{
 "Text": "Amazon Translate translates documents between languages in
 real time. It uses advanced machine learning technologies
 to provide high-quality real-time translation. Use it to
 translate documents or to build applications that work in
 multiple languages.",
 "SourceLanguageCode": "en",
 "TargetLanguageCode": "fr"
}

aws translate translate-text \
 --region region \
 --cli-input-json file://translate.json > translated.json
 
 Result: A French Translation hould be displayed
 
 Code: 
 	 
	 func translate(text: String) {
	 var credentialsProvider = AWSStaticCredentialsProvider(accessKey: "access key", secretKey:
	 "secret key")
	var configuration = AWSServiceConfiguration(region: AWSRegionUSEast1, credentialsProvider:
	 credentialsProvider)

	 AWSServiceManager.default().defaultServiceConfiguration = configuration
	let translateClient = AWSTranslate.default()
	let translateRequest = AWSTranslateTranslateTextRequest()
	translateRequest?.sourceLanguageCode = "en"
	translateRequest?.targetLanguageCode = "es"
	translateRequest?.text = string

	let callback: (AWSTranslateTranslateTextResponse?, Error?) -> Void = { (response, error) in
	 guard let response = response else {
	 print("Got error \(error)")
	 return
	 }

	 if let translatedText = response.translatedText {
	 print(translatedText)
	 }
	}

	translateClient.translateText(translateRequest!, completionHandler: callback)


6. Use Aws Comprehend's sentiment detection in Swift

		func example() {

		var text = "It is raining today in Seattle";

		// Create credentials using a provider chain. For more information, see
		// https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html
			AWSCredentialsProvider awsCreds = DefaultAWSCredentialsProviderChain.getInstance();

			AmazonComprehend comprehendClient =
		   	 AmazonComprehendClientBuilder.standard()
						 .withCredentials(awsCreds)
						 .withRegion("region")
						 .build();

		// Call detectSentiment API
		print("Calling DetectSentiment");
		DetectSentimentRequest detectSentimentRequest = new DetectSentimentRequest().withText(text)
											    .withLanguageCode("en");
		DetectSentimentResult detectSentimentResult = comprehendClient.detectSentiment(detectSentimentRequest);
		print(detectSentimentResult);

	    }

	}







