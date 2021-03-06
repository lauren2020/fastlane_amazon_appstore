fastlane_require 'httparty'
default_platform(:android)

###################################################
############       CONFIGURATION       ############
###################################################
$path_to_apk = [PATH_TO_RELEASE_APK]

# Amazon API Variables #
$amazon_developer_base_url = "https://developer.amazon.com/api/appstore"
$amazon_auth_url = "https://api.amazon.com/auth/O2/token"
$amazon_api_version = "v1"
$amazon_app_id = [AMAZON_APP_ID]
$amazon_client_id = [AMAZON_CLIENT_ID]
$amazon_client_secret = [AMAZON_CLIENT_SECRET]

$release_notes = [STATIC_RELEASE_NOTES]
###################################################

platform :android do

  desc "Deploy a new version to the Amazon App Store"
  lane :deploy do
    release_to_amazon
  end

  desc "Deploy to the Amazon app store"
  private_lane :release_to_amazon do

    ##
    # Access Token
    ##
    puts "Get access token"
    response = HTTParty.post($amazon_auth_url, { headers: { 'Content-Type' => 'application/x-www-form-urlencoded' }, body: { grant_type: "client_credentials", client_id: $amazon_client_id, client_secret: $amazon_client_secret, scope: "appstore::apps:readwrite" }})
    access_token = response["access_token"]

    ##
    # Create Edit
    ##
    UI.success "Get edit id"
    response = HTTParty.get("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }})
    UI.success response
    edit_status = response["status"]
    UI.success edit_status

    if edit_status == "IN_PROGRESS"
      UI.success "Using current release"
      edit_id = response["id"]
    elsif edit_status == "REVIEW"
      error(error_message:  "Amazon app store rejected the submission because an app is already under review for release.  Release to the Amazon store manually.")
      return
    else
      UI.success "Creating new release"
      edit_id = HTTParty.post("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }})["id"]
    end

    UI.success "Edit id: "+ edit_id

    ##
    # Get Current APK
    ##
    UI.success "Getting current APK information"
    response = HTTParty.get("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/apks", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }})
    apk_id = response[0]["id"]
    UI.success "APK id: " + apk_id

    ##
    # Replace APK
    ##
    UI.success "Uploading APK"
    response = HTTParty.get("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/apks/#{apk_id}", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }})
    etag = response.headers["ETag"]
    UI.success "ETag: " + etag
    response = HTTParty.put("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/apks/#{apk_id}/replace", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => 'application/octet-stream', 'Accept' => 'application/json', 'If-Match' => etag}, body: File.new($path_to_apk, 'rb').read})
    UI.success "Response: " + response.to_s

    ##
    # Update Release Notes
    ##
    UI.success "Updating release notes"
    response = HTTParty.get("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/listings", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }})
    etag = response.headers["ETag"]
    listing = response["listings"]["en-US"]
    listing["recentChanges"] = $release_notes
    UI.success listing.to_s
    HTTParty.put("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/listings/en-US", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => 'application/json', 'If-Match' => etag}, body: listing.to_json})

    ##
    # Validate & Commit Edit
    ##
    etag = HTTParty.get("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}", { headers: { 'Authorization' => "Bearer #{access_token}", 'Content-Type' => "application/json" }}).headers["ETag"]
    response = HTTParty.post("#{$amazon_developer_base_url}/#{$amazon_api_version}/applications/#{$amazon_app_id}/edits/#{edit_id}/commit", { headers: { 'Authorization' => "Bearer #{access_token}", 'If-Match' => etag }})

    if response.code != 200
      #Handle error with validation
      error(error_message: response.to_s)
    else
      #Successfully deployed
      UI.success "SUCCESS"
    end
  end
end
