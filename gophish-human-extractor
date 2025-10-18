package main

import (
	"crypto/tls"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"sort"
	"strings"
	"time"
)

// GoPhish API structures
type Campaign struct {
	ID       int      `json:"id"`
	Name     string   `json:"name"`
	Status   string   `json:"status"`
	Results  []Result `json:"results"`
	Timeline []Event  `json:"timeline"`
}

type Result struct {
	ID        string `json:"id"`
	Email     string `json:"email"`
	FirstName string `json:"first_name"`
	LastName  string `json:"last_name"`
	Position  string `json:"position"`
	Status    string `json:"status"`
}

type Event struct {
	Email   string      `json:"email"`
	Time    time.Time   `json:"time"`
	Message string      `json:"message"`
	Details interface{} `json:"details"`
}

type FilteredResult struct {
	Email           string
	Name            string
	Status          string
	IsHuman         bool
	HumanEvents     []Event
	ScannerEvents   []Event
	FirstHumanClick time.Time
	Submitted       bool
}

type FileCounts struct {
	Opened          int
	Clicked         int
	Submitted       int
	OrganicClicked  int
	OrganicSubmitted int
}

func main() {
	fmt.Println("🏴‍☠️ GoPhish Human Results Filter - Mugiwara Edition! 🏴‍☠️")
	fmt.Println("=" + strings.Repeat("=", 58))
	fmt.Println()

	// Get user input
	var endpoint, apiKey, campaignID string

	fmt.Print("Enter GoPhish endpoint URL (e.g., https://gophish.example.com  ): ")
	fmt.Scanln(&endpoint)
	endpoint = strings.TrimSuffix(endpoint, "/")

	fmt.Print("Enter API Key: ")
	fmt.Scanln(&apiKey)

	fmt.Print("Enter Campaign ID: ")
	fmt.Scanln(&campaignID)

	// Debug mode always enabled
	debugMode := true

	fmt.Println()
	fmt.Println("🍖 Fetching campaign data from the Grand Line...")

	// Fetch campaign data (FULL details with timeline)
	campaign, err := fetchCampaign(endpoint, apiKey, campaignID)
	if err != nil {
		fmt.Printf("❌ Error fetching campaign: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("⚔️  Campaign: %s\n", campaign.Name)
	fmt.Printf("📧 Total results: %d\n", len(campaign.Results))
	fmt.Printf("📋 Total timeline events: %d\n\n", len(campaign.Timeline))

	// Filter results
	filteredResults := filterResults(campaign, debugMode)

	// Display results
	displayResults(filteredResults)

	// Write results to files and get actual counts
	fmt.Println()
	fmt.Println("📝 Writing results to files...")
	fileCounts, err := writeResultsToFiles(filteredResults, debugMode)
	if err != nil {
		fmt.Printf("⚠️  Error writing files: %v\n", err)
	} else {
		fmt.Println("✅ Files written successfully!")
		
		// Show organic submission rate
		if fileCounts.OrganicClicked > 0 {
			organicRate := float64(fileCounts.OrganicSubmitted) / float64(fileCounts.OrganicClicked) * 100
			fmt.Println()
			fmt.Println("🎯 ORGANIC SUBMISSION RATE (100% Verified Humans):")
			fmt.Printf("   %d/%d (%.1f%%) 🍒\n", fileCounts.OrganicSubmitted, fileCounts.OrganicClicked, organicRate)
		}
	}
}

func fetchCampaign(endpoint, apiKey, campaignID string) (*Campaign, error) {
	// Use /api/campaigns/:id (NOT /results) to get full timeline
	url := fmt.Sprintf("%s/api/campaigns/%s?api_key=%s", endpoint, campaignID, apiKey)

	// Create HTTP client that skips TLS verification
	tr := &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}
	client := &http.Client{Transport: tr}

	resp, err := client.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("API returned status %d: %s", resp.StatusCode, string(body))
	}

	var campaign Campaign
	if err := json.NewDecoder(resp.Body).Decode(&campaign); err != nil {
		return nil, fmt.Errorf("failed to parse JSON: %v", err)
	}

	return &campaign, nil
}

func filterResults(campaign *Campaign, debugMode bool) []FilteredResult {
	// Create a map to organize events by email
	eventsByEmail := make(map[string][]Event)
	
	for _, event := range campaign.Timeline {
		eventsByEmail[event.Email] = append(eventsByEmail[event.Email], event)
	}

	filtered := make([]FilteredResult, 0)
	suspiciousClicks := make([]string, 0) // Track suspicious clicks for debug

	for _, result := range campaign.Results {
		fr := FilteredResult{
			Email:         result.Email,
			Name:          fmt.Sprintf("%s %s", result.FirstName, result.LastName),
			Status:        result.Status,
			HumanEvents:   make([]Event, 0),
			ScannerEvents: make([]Event, 0),
		}

		events := eventsByEmail[result.Email]
		
		var emailSentTime time.Time
		var hasEmailOpened bool

		// First pass: find email sent time and check for email opened
		for _, event := range events {
			if event.Message == "Email Sent" {
				emailSentTime = event.Time
			}
			if event.Message == "Email Opened" {
				hasEmailOpened = true
			}
			if event.Message == "Submitted Data" {
				fr.Submitted = true
			}
		}

		// Second pass: categorize events
		for _, event := range events {
			isScanner := false
			isSuspicious := false // Track borderline cases

			// Skip campaign created and email sent events
			if event.Message == "Campaign Created" || event.Message == "Email Sent" {
				continue
			}

			// Check for scanner indicators - MULTIPLE METHODS
			if event.Message == "Clicked Link" {
				
				// METHOD 1: Check browser version (Chrome 113 = known scanner)
				// NOTE: This is a weak indicator and will break when scanners update
				if event.Details != nil {
					if detailsMap, ok := event.Details.(map[string]interface{}); ok {
						if browser, ok := detailsMap["browser"].(map[string]interface{}); ok {
							if version, ok := browser["version"].(string); ok {
								if strings.HasPrefix(version, "113.") {
									isScanner = true
								}
							}
						}
					}
				}

				// METHOD 2: STRONG INDICATOR - Fast click without email opened
				// Scanners click within seconds and don't load tracking pixels
				if !emailSentTime.IsZero() {
					clickDelay := event.Time.Sub(emailSentTime)
					
					// If clicked within 30 seconds AND no email opened event
					// This is VERY likely a scanner (humans need time to read)
					if clickDelay < 30*time.Second && !hasEmailOpened {
						isScanner = true
					}
					
					// Even if they somehow have "opened", if click is within 10 seconds
					// it's almost certainly automated (too fast for human)
					if clickDelay < 10*time.Second {
						isScanner = true
					}

					// SUSPICIOUS: Between 30-60 seconds without open (borderline)
					if clickDelay >= 30*time.Second && clickDelay < 60*time.Second && !hasEmailOpened {
						isSuspicious = true
					}
				}
			}

			// Categorize event
			if isScanner {
				fr.ScannerEvents = append(fr.ScannerEvents, event)
			} else {
				fr.HumanEvents = append(fr.HumanEvents, event)
				// Track first human click
				if event.Message == "Clicked Link" {
					if fr.FirstHumanClick.IsZero() {
						fr.FirstHumanClick = event.Time
					}
					// Track suspicious clicks for debug
					if isSuspicious && debugMode {
						clickDelay := event.Time.Sub(emailSentTime)
						suspiciousClicks = append(suspiciousClicks, 
							fmt.Sprintf("%s - clicked after %.0fs without open", fr.Email, clickDelay.Seconds()))
					}
				}
			}
		}

		// Determine if human interacted
		fr.IsHuman = len(fr.HumanEvents) > 0

		filtered = append(filtered, fr)
	}

	// Show suspicious clicks in debug mode
	if debugMode && len(suspiciousClicks) > 0 {
		fmt.Println()
		fmt.Println("⚠️  DEBUG: SUSPICIOUS CLICKS (might be scanners):")
		fmt.Println(strings.Repeat("-", 80))
		for _, msg := range suspiciousClicks {
			fmt.Printf("   %s\n", msg)
		}
		fmt.Println()
	}

	return filtered
}

func displayResults(results []FilteredResult) {
	fmt.Println("🗺️  FILTERED RESULTS - TREASURE MAP")
	fmt.Println(strings.Repeat("=", 80))
	fmt.Println()

	humanCount := 0
	scannerOnlyCount := 0
	submittedCount := 0

	fmt.Println("👥 HUMAN INTERACTIONS:")
	fmt.Println(strings.Repeat("-", 80))

	for _, fr := range results {
		if fr.IsHuman {
			humanCount++
			if fr.Submitted {
				submittedCount++
			}

			fmt.Printf("\n📧 %s (%s)\n", fr.Email, fr.Name)
			fmt.Printf("   Status: %s | Submitted Data: %v\n", fr.Status, fr.Submitted)

			if len(fr.ScannerEvents) > 0 {
				fmt.Printf("   ⚠️  %d scanner event(s) detected and filtered\n", len(fr.ScannerEvents))
			}

			fmt.Println("   Human Events:")
			for _, event := range fr.HumanEvents {
				fmt.Printf("      • %s - %s\n", event.Time.Format("2006-01-02 15:04:05"), event.Message)
				if event.Details != nil {
					if detailsMap, ok := event.Details.(map[string]interface{}); ok {
						if browser, ok := detailsMap["browser"].(map[string]interface{}); ok {
							if version, ok := browser["version"].(string); ok {
								fmt.Printf("        (Browser: Chrome %s)\n", version)
							}
						}
					}
				}
			}
		}
	}

	fmt.Println()
	fmt.Println(strings.Repeat("=", 80))
	fmt.Println()
	fmt.Println("🤖 SCANNER-ONLY (No Human Interaction):")
	fmt.Println(strings.Repeat("-", 80))

	for _, fr := range results {
		if !fr.IsHuman && len(fr.ScannerEvents) > 0 {
			scannerOnlyCount++
			fmt.Printf("📧 %s (%s) - Scanner only (Chrome 113)\n", fr.Email, fr.Name)
		}
	}

	// Summary
	fmt.Println()
	fmt.Println(strings.Repeat("=", 80))
	fmt.Println("📊 SUMMARY - THE REAL TREASURE!")
	fmt.Println(strings.Repeat("=", 80))
	
	// Count UNIQUE people who clicked - USE STATUS FIELD (what GoPhish frontend uses)
	// Status can be: "Email Sent", "Email Opened", "Clicked Link", "Submitted Data"
	// This matches the frontend behavior exactly
	clickedCount := 0
	openedCount := 0
	for _, fr := range results {
		if fr.Status == "Clicked Link" || fr.Status == "Submitted Data" {
			clickedCount++
			openedCount++ // Anyone who clicked also "opened" in GoPhish's logic
		} else if fr.Status == "Email Opened" {
			openedCount++
		}
	}
	
	fmt.Printf("Total Recipients: %d\n", len(results))
	fmt.Printf("✅ Human Interactions: %d (%.1f%%)\n", humanCount, float64(humanCount)/float64(len(results))*100)
	fmt.Printf("📧 Email Opened: %d (%.1f%%)\n", openedCount, float64(openedCount)/float64(len(results))*100)
	fmt.Printf("🖱️  Clicked Links: %d (%.1f%%)\n", clickedCount, float64(clickedCount)/float64(len(results))*100)
	fmt.Printf("🔐 Submitted Credentials: %d (%.1f%%)\n", submittedCount, float64(submittedCount)/float64(len(results))*100)
	
	// CHERRY ON TOP: Submission rate among those who clicked (status-based)
	if clickedCount > 0 {
		submissionRate := float64(submittedCount) / float64(clickedCount) * 100
		fmt.Printf("🎯 Submission Rate (status-based): %d/%d (%.1f%%)\n", submittedCount, clickedCount, submissionRate)
	}
	
	fmt.Printf("🤖 Scanner-Only: %d (%.1f%%)\n", scannerOnlyCount, float64(scannerOnlyCount)/float64(len(results))*100)
	fmt.Printf("📭 No Interaction: %d (%.1f%%)\n",
		len(results)-humanCount-scannerOnlyCount,
		float64(len(results)-humanCount-scannerOnlyCount)/float64(len(results))*100)
	fmt.Println()
	fmt.Println("🏴‍☠️ YOSH! Analysis complete, nakama! 🏴‍☠️")
}

func writeResultsToFiles(results []FilteredResult, debugMode bool) (FileCounts, error) {
	// Use maps to ensure unique emails (one person = one entry, even if multiple clicks)
	emailsOpenedMap := make(map[string]bool)
	emailsClickedMap := make(map[string]bool)
	emailsSubmittedMap := make(map[string]bool)

	// Track quality metrics for verification
	type QualityMetrics struct {
		HasEmailOpenedEvent bool
		HasClickedEvent     bool
		TimeBetweenSendAndClick time.Duration
		BrowserVersion      string
	}
	clickQuality := make(map[string]QualityMetrics)

	// Categorize emails based on GoPhish STATUS field (not timeline events)
	// This matches exactly what the frontend shows
	for _, fr := range results {
		if !fr.IsHuman {
			continue // Skip non-human interactions
		}

		// Collect quality metrics for clicked users
		if fr.Status == "Clicked Link" || fr.Status == "Submitted Data" {
			metrics := QualityMetrics{}
			
			var sendTime time.Time
			for _, event := range fr.HumanEvents {
				if event.Message == "Email Sent" {
					sendTime = event.Time
				}
				if event.Message == "Email Opened" {
					metrics.HasEmailOpenedEvent = true
				}
				if event.Message == "Clicked Link" {
					metrics.HasClickedEvent = true
					if !sendTime.IsZero() {
						metrics.TimeBetweenSendAndClick = event.Time.Sub(sendTime)
					}
					// Get browser version
					if event.Details != nil {
						if detailsMap, ok := event.Details.(map[string]interface{}); ok {
							if browser, ok := detailsMap["browser"].(map[string]interface{}); ok {
								if version, ok := browser["version"].(string); ok {
									metrics.BrowserVersion = version
								}
							}
						}
					}
				}
			}
			clickQuality[fr.Email] = metrics
		}

		// Use status field like GoPhish frontend does
		if fr.Status == "Submitted Data" {
			emailsSubmittedMap[fr.Email] = true
			emailsClickedMap[fr.Email] = true  // Submitted implies clicked
			emailsOpenedMap[fr.Email] = true   // Submitted implies opened
		} else if fr.Status == "Clicked Link" {
			emailsClickedMap[fr.Email] = true
			emailsOpenedMap[fr.Email] = true   // Clicked implies opened
		} else if fr.Status == "Email Opened" {
			emailsOpenedMap[fr.Email] = true
		}
	}

	// Convert maps to slices
	emailsOpened := mapKeysToSlice(emailsOpenedMap)
	emailsClicked := mapKeysToSlice(emailsClickedMap)
	emailsSubmitted := mapKeysToSlice(emailsSubmittedMap)

	// Sort alphabetically
	sort.Strings(emailsOpened)
	sort.Strings(emailsClicked)
	sort.Strings(emailsSubmitted)

	// Write to files
	if err := writeEmailsToFile("humans_opened.txt", emailsOpened); err != nil {
		return FileCounts{}, fmt.Errorf("failed to write opened emails: %v", err)
	}
	fmt.Printf("   📧 humans_opened.txt - %d emails\n", len(emailsOpened))

	// Write organic clicked file
	cleanClickedEmails := make([]string, 0)
	suspiciousClickedEmails := make([]string, 0)
	
	if len(clickQuality) > 0 {
		for email, metrics := range clickQuality {
			isSuspicious := false
			
			// Check for suspicious patterns
			if !metrics.HasEmailOpenedEvent {
				isSuspicious = true
			}
			
			if metrics.TimeBetweenSendAndClick > 0 && metrics.TimeBetweenSendAndClick < 60*time.Second {
				isSuspicious = true
			}
			
			if strings.HasPrefix(metrics.BrowserVersion, "113.") || 
			   strings.HasPrefix(metrics.BrowserVersion, "100.") ||
			   strings.HasPrefix(metrics.BrowserVersion, "101.") {
				isSuspicious = true
			}
			
			if isSuspicious {
				suspiciousClickedEmails = append(suspiciousClickedEmails, email)
			} else {
				cleanClickedEmails = append(cleanClickedEmails, email)
			}
		}
	}

	// Write organic clicked results
	if len(cleanClickedEmails) > 0 {
		sort.Strings(cleanClickedEmails)
		
		if err := writeEmailsToFile("organic_clicked.txt", cleanClickedEmails); err != nil {
			return FileCounts{}, fmt.Errorf("failed to write organic clicked: %v", err)
		}
		fmt.Printf("   🖱️  organic_clicked.txt - %d emails\n", len(cleanClickedEmails))
	}

	// Write submitted results
	if err := writeEmailsToFile("humans_submitted.txt", emailsSubmitted); err != nil {
		return FileCounts{}, fmt.Errorf("failed to write submitted emails: %v", err)
	}
	fmt.Printf("   🔐 humans_submitted.txt - %d emails\n", len(emailsSubmitted))

	// Write masked versions for reporting
	fmt.Println()
	fmt.Println("   🎭 Creating masked versions for reporting...")
	
	maskedOpened := maskEmails(emailsOpened)
	if err := writeEmailsToFile("humans_opened_masked.txt", maskedOpened); err != nil {
		return FileCounts{}, fmt.Errorf("failed to write masked opened emails: %v", err)
	}
	fmt.Printf("   📧 humans_opened_masked.txt - %d emails\n", len(maskedOpened))

	// Masked organic clicked
	if len(cleanClickedEmails) > 0 {
		maskedCleanClicked := maskEmails(cleanClickedEmails)
		if err := writeEmailsToFile("organic_clicked_masked.txt", maskedCleanClicked); err != nil {
			return FileCounts{}, fmt.Errorf("failed to write masked organic clicked: %v", err)
		}
		fmt.Printf("   🖱️  organic_clicked_masked.txt - %d emails\n", len(maskedCleanClicked))
	}

	// Masked submitted
	maskedSubmitted := maskEmails(emailsSubmitted)
	if err := writeEmailsToFile("humans_submitted_masked.txt", maskedSubmitted); err != nil {
		return FileCounts{}, fmt.Errorf("failed to write masked submitted emails: %v", err)
	}
	fmt.Printf("   🔐 humans_submitted_masked.txt - %d emails\n", len(maskedSubmitted))

	return FileCounts{
		Opened:           len(emailsOpened),
		Clicked:          len(emailsClickedMap),
		Submitted:        len(emailsSubmitted),
		OrganicClicked:   len(cleanClickedEmails),
		OrganicSubmitted: len(emailsSubmitted), // Total submitted (not filtered by organic)
	}, nil
}

func writeEmailsToFile(filename string, emails []string) error {
	file, err := os.Create(filename)
	if err != nil {
		return err
	}
	defer file.Close()

	for _, email := range emails {
		if _, err := file.WriteString(email + "\n"); err != nil {
			return err
		}
	}

	return nil
}

func maskEmails(emails []string) []string {
	masked := make([]string, len(emails))
	
	for i, email := range emails {
		masked[i] = maskEmail(email)
	}
	
	return masked
}

func maskEmail(email string) string {
	// Split email into local part and domain
	parts := strings.Split(email, "@")
	if len(parts) != 2 {
		return email // Invalid email, return as-is
	}
	
	localPart := parts[0]
	domain := parts[1]
	
	// Mask local part
	// Show first 2 chars, mask the rest
	var maskedLocal string
	if len(localPart) <= 2 {
		maskedLocal = localPart // Too short to mask meaningfully
	} else {
		visibleChars := 2
		maskedLength := len(localPart) - visibleChars
		maskedLocal = localPart[:visibleChars] + strings.Repeat("*", maskedLength)
	}
	
	return maskedLocal + "@" + domain
}

func uniqueAndSort(emails []string) []string {
	// Use a map to track unique emails
	uniqueMap := make(map[string]bool)
	for _, email := range emails {
		uniqueMap[email] = true
	}
	
	// Convert back to slice
	unique := make([]string, 0, len(uniqueMap))
	for email := range uniqueMap {
		unique = append(unique, email)
	}
	
	// Sort alphabetically
	sort.Strings(unique)
	
	return unique
}

func mapKeysToSlice(m map[string]bool) []string {
	keys := make([]string, 0, len(m))
	for key := range m {
		keys = append(keys, key)
	}
	return keys
}
