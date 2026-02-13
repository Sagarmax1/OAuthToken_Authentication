# OAuthToken_Authentication
In this Repository , i used to Generte OAuth Token and Try to send PayLoad from Console Appplication


 static async Task Main(string[] args)
        {
            Console.WriteLine("Token Generating...");

            string clientId = "2ef2b124-4106-4a9b-9b7d-009041b98c1f";
            string clientSecret = "H-qaLCz9oXdHgZZ8MO6pPk6KUBeeCWpLY73uN6hVmKc";

            // HardCoaded Value for passing overridding port issue  address
            string correctIp = "10.2.6.74";
            string hostName = "nda-hclt-poq.hclt.corp.hcl.in";
            string path = "/RESTAdapter/OAuthServer";
            var uri = new Uri($"https://{correctIp}:50001{path}");

            //   Force TLS 1.2(required) Transport layer security
            ServicePointManager.SecurityProtocol = SecurityProtocolType.Tls12;

            // HarCoded for passing Server issue by sagar
            ServicePointManager.ServerCertificateValidationCallback = delegate { return true; };

            using (var handler = new HttpClientHandler())
            using (var http = new HttpClient(handler))
            {
                http.Timeout = TimeSpan.FromSeconds(30);

                var basic = Convert.ToBase64String(Encoding.ASCII.GetBytes(clientId + ":" + clientSecret));
                http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", basic);

                http.DefaultRequestHeaders.Host = hostName; // Hard coded for hostName by sagar 

                var form = new Dictionary<string, string>
                {
                    ["grant_type"] = "client_credentials"
                };

                using (var content = new FormUrlEncodedContent(form))
                {
                    HttpResponseMessage response = null;

                    try
                    {
                        response = await http.PostAsync(uri, content);
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine("Error calling token endpoint: " + ex.Message);
                        return;
                    }

                    string body = await response.Content.ReadAsStringAsync();

                    Console.WriteLine("\nCorporate Token Response:");
                    Console.WriteLine(body);
                    Console.WriteLine("--------------------------------------");

                    // Parse token and enforce 1-day validity in the app
                    string accessToken = null;
                    DateTime utcNow = DateTime.UtcNow;
                    DateTime expiresAtUtc = utcNow.AddSeconds(86400); // exactly 1 day

                    try
                    {
                        // Use Newtonsoft to parse JSON
                        var json = JObject.Parse(body);

                        JToken at;
                        if (json.TryGetValue("access_token", StringComparison.OrdinalIgnoreCase, out at))
                        {
                            accessToken = (string)at;
                        }

                        if (!string.IsNullOrEmpty(accessToken))
                        {
                            Console.WriteLine("Token Generated Successfully!");
                            Console.WriteLine("Access Token (redacted): " + Redact(accessToken));
                            Console.WriteLine("Effective Expiry (UTC): " + expiresAtUtc.ToString("u") + " (1 day from now)");
                            Console.WriteLine();
                            Console.WriteLine("Action: Request a NEW token after this time.");

                            // Payload Code Below
                            http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", accessToken);

                            string apiPath = "/RESTAdapter/Sys_SOW/oa_SOW_Automation_Request";
                            var apiUri = new Uri($"https://{correctIp}:50001{apiPath}");

                            http.DefaultRequestHeaders.Host = hostName;
         
                            string payloadJson = @"
                            {
                             ""oa_SOW_Automation_Request"": {
                                                  ""DATA"": 
                            {
                                ""SOW_ID"": ""291193"",
                                ""Vendor_Code"": ""2400120860"",
                                ""SOW_Start_Date"": ""2026-01-08"",
                                ""SOW_End_Date"": ""2026-07-03"",
                                ""Reference_No"": ""1"",
                                ""CONS_NAME"": ""Gopal Sahu"",
                                ""Vendor_Name"": ""PARKAR DIGITAL PTE LTD [2400120860]"",
                                ""Vendor_Email_ID"": ""contracts@parkar.in;tpsow@hcltech.com;qashrina.quzaima@hcltech.com;yap.ni@hcltech.com"",
                                ""RM_ID"": ""51813979"",
                                ""RM_Name"": ""Umesha Sanganakal Guntaga"",
                                ""RM_Email_ID"": ""UMESHASANGANA.GUNTA@HCLTECH.COM"",
                                ""Client_Name"": ""Gilead Sciences Inc"",
                                ""Project_Name"": ""Gilead SAP APAC Rollout"",
                                ""Location"": ""Singapore"",
                                ""Skill"": ""Project Management Skills (APPS)-Program Management-Program Management"",
                                ""Relevant_Experience"": ""324"",
                                ""Total_Experience"": ""324"",
                                ""Resale_Services"": ""2"",
                                ""Buy_Rate"": ""18437.33"",
                                ""Currency"": ""SGD"",
                               ""Wage_Type"": ""Monthly Rate"",
                               ""OT_Rate"": ""0"",
                               ""Currency_OT"": ""NA"",
                               ""Wage_Type_OT"": ""NA"",
                               ""DT_Rate"": ""0"",
                               ""Currency_DT"": ""SGD"",
                               ""Wage_Type_DT"": ""Monthly Rate"",
                               ""Consultant_SAP_ID"": ""52377234"",
                               ""Other_Conditions"": ""Includes 9% GST etc."",
                               ""SOW_Created_By"": ""51981983"",
                               ""Changed_By"": ""51981983"",
                               ""Project_Specific_Details"": """",
                               ""ZZ_SOW_BACKDATE_EFFECTIVE"": """",
                               ""ZZ_ADDI_BUY_RATE"": """"
                            }
                            
                        }
                }";

                            using (var apiContent = new StringContent(payloadJson, Encoding.UTF8, "application/json"))
                            {
                                HttpResponseMessage apiResponse = null;
                                try
                                {
                                    apiResponse = await http.PostAsync(apiUri, apiContent);
                                }
                                catch (Exception exApi)
                                {
                                    Console.WriteLine("\nError calling target API: " + exApi.Message);
                                    return;
                                }

                                string apiBody = await apiResponse.Content.ReadAsStringAsync();

                                Console.WriteLine("\n=== Target API Response ===");
                                Console.WriteLine("HTTP " + (int)apiResponse.StatusCode + " " + apiResponse.ReasonPhrase);
                                Console.WriteLine(apiBody);

                                if (!apiResponse.IsSuccessStatusCode)
                                {
                                    Console.WriteLine("\nAPI call failed. Please review the response above.");
                                }
                                else
                                {
                                    Console.WriteLine("\nAPI call succeeded.");
                                }
                            }
                        }
                        else
                        {
                            Console.WriteLine("Token Generation Failed! 'access_token' not found in response.");
                        }
                    }
                    catch (Exception)
                    {
                        Console.WriteLine("Failed to parse token response as JSON.");
                    }
                    finally
                    {
                        http.Dispose();
                        handler.Dispose();
                    }
                }
            }

            Console.WriteLine("\nPress ENTER to exit...");
            Console.ReadLine();
        }

        // Helper Method for token length 
        static string Redact(string token)
        {
            if (string.IsNullOrEmpty(token)) return token;
            if (token.Length <= 12) return "********";
            return token.Substring(0, 6) + "..." + token.Substring(token.Length - 4);
        }
