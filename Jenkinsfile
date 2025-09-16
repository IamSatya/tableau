pipeline {
    agent any

    environment {
        TABLEAU_SERVER    = 'https://prod-apsoutheast-b.online.tableau.com'
        TABLEAU_SITE_URL  = 'satyanarayanap-45d27fc93a'
        GIT_PROJECTS_DIR  = 'workbooks'
        LOGFILE           = 'log.txt'
        GIT_CHANGES_FILE  = 'git_file_changes.txt'
        GIT_CONTENT_DIFF_FILE = 'git_filetext_changes.txt'
    }

    options {
        ansiColor('xterm')
        timestamps()
    }

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: params.GIT_BRANCH, url: 'https://your.git.repo/url.git'
            }
        }

        stage('Install Tools') {
            steps {
                sh '''
                if ! command -v jq >/dev/null 2>&1; then echo "Installing jq..." && sudo apt-get update && sudo apt-get install -y jq; fi
                if ! command -v unzip >/dev/null 2>&1; then echo "Installing unzip..." && sudo apt-get install -y unzip; fi
                if ! command -v openssl >/dev/null 2>&1; then echo "Installing openssl..." && sudo apt-get install -y openssl; fi
                '''
            }
        }

        stage('Run Tableau Sync') {
            environment {
                TABLEAU_USER = credentials('tableau-username')  // Jenkins ID storing Tableau username
                TABLEAU_PW   = credentials('tableau-password')  // Jenkins ID storing Tableau password
            }
            steps {
                sh '''
#!/bin/bash
set -euo pipefail
> "$LOGFILE"

# ====================================================================================
# Config (from Jenkins env)
# ====================================================================================
TABLEAU_SERVER="$TABLEAU_SERVER"
TABLEAU_USER="$TABLEAU_USER"
TABLEAU_PW="$TABLEAU_PW"
TABLEAU_SITE_URL="$TABLEAU_SITE_URL"
GIT_PROJECTS_DIR="$GIT_PROJECTS_DIR"
LOGFILE="$LOGFILE"
GIT_CHANGES_FILE="$GIT_CHANGES_FILE"
GIT_CONTENT_DIFF_FILE="$GIT_CONTENT_DIFF_FILE"

# Ignored projects
IGNORED_PROJECTS=("default" "Samples" "External Assets Default Project")

# ====================================================================================
# Helpers
# ====================================================================================
contains_element() { local e match="$1"; shift; for e; do [[ "$e" == "$match" ]] && return 0; done; return 1; }

# version compare using sort -V (returns 0 if $1 <= $2)
ver_lte() { [[ "$(printf '%s\\n%s\\n' "$1" "$2" | sort -V | head -n1)" == "$1" ]]; }

# Extract Tableau workbook version from a .twb or .twbx
extract_workbook_version() {
  local path="$1"
  local ext="${path##*.}"
  local xml_line=""
  if [[ "$ext" == "twb" ]]; then
    # read the line with <workbook ... version="x.x">
    xml_line=$(grep -m1 -E '<workbook[[:space:]][^>]*version=' "$path" || true)
  elif [[ "$ext" == "twbx" ]]; then
    local tmpdir; tmpdir=$(mktemp -d)
    trap 'rm -rf "$tmpdir"' RETURN
    unzip -qq "$path" -d "$tmpdir" || true
    local twb; twb=$(find "$tmpdir" -type f -name "*.twb" | head -n1)
    if [[ -n "$twb" && -f "$twb" ]]; then
      xml_line=$(grep -m1 -E '<workbook[[:space:]][^>]*version=' "$twb" || true)
    fi
  fi
  # Parse version="X.Y[.Z]" from the line
  if [[ -n "$xml_line" ]]; then
    echo "$xml_line" | sed -n 's/.*version="\\([^"]*\\)".*/\\1/p' | head -n1
  fi
}

# Refactored publish logic into a single function
publish_workbook() {
  local file_to_publish="$1"
  local pub_project_name; pub_project_name=$(basename "$(dirname "$file_to_publish")")
  local pub_workbook_filename; pub_workbook_filename=$(basename "$file_to_publish")
  local pub_workbook_name_no_ext; pub_workbook_name_no_ext="${pub_workbook_filename%.*}"

  # On-demand project creation
  if [[ -z "${tableau_projects[$pub_project_name]+_}" ]]; then
    echo "  CREATE: Project '$pub_project_name' does not exist. Attempting to create..." | tee -a "$LOGFILE"
    CREATE_RESPONSE=$(curl -s -X POST "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/projects" \
      -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Tableau-Auth: $TOKEN" \
      -d "{\"project\": {\"name\": \"$pub_project_name\"}}")

    if ! [[ "$CREATE_RESPONSE" == "{"* ]]; then
      echo "    ERROR: Non-JSON response while creating project '$pub_project_name'. Skipping publish." | tee -a "$LOGFILE"
      echo "    DEBUG: Server Response: $CREATE_RESPONSE" | tee -a "$LOGFILE"
      return 1
    fi
    if echo "$CREATE_RESPONSE" | jq -e '.error' >/dev/null; then
      echo "    ERROR: Tableau API error while creating '$pub_project_name'." | tee -a "$LOGFILE"
      echo "    DEBUG: $(echo "$CREATE_RESPONSE" | jq .)" | tee -a "$LOGFILE"
      return 1
    else
      local new_id; new_id=$(echo "$CREATE_RESPONSE" | jq -r '.project.id')
      tableau_projects["$pub_project_name"]="$new_id"
      echo "    SUCCESS: Created project '$pub_project_name' with ID $new_id." | tee -a "$LOGFILE"
    fi
  fi

  local project_id=${tableau_projects[$pub_project_name]}
  local workbook_type="${file_to_publish##*.}"
  local boundary="------------------------$(openssl rand -hex 16)"

  ( printf -- "--%s\\r\\n" "$boundary"
    printf "Content-Disposition: name=\\"request_payload\\"\\r\\n\\r\\n<tsRequest><workbook name=\\"%s\\" showTabs=\\"false\\"><project id=\\"%s\\"/></workbook></tsRequest>\\r\\n" "$pub_workbook_name_no_ext" "$project_id"
    printf -- "--%s\\r\\n" "$boundary"
    printf "Content-Disposition: name=\\"tableau_workbook\\"; filename=\\"%s\\"\\r\\n\\r\\n" "$pub_workbook_filename"
    cat "$file_to_publish"
    printf "\\r\\n--%s--\\r\\n" "$boundary"
  ) > publish_body.bin

  PUBLISH_RESPONSE=$(curl -s -X POST "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/workbooks?workbookType=$workbook_type&overwrite=true" \
      -H "Content-Type: multipart/mixed; boundary=$boundary" \
      -H "X-Tableau-Auth: $TOKEN" \
      --data-binary @publish_body.bin)
  rm -f publish_body.bin

  if [[ $PUBLISH_RESPONSE == *"<error"* ]]; then
    echo "    FINAL STATUS: ERROR publishing '$pub_workbook_filename'." | tee -a "$LOGFILE"
    echo "    DEBUG: Response: $PUBLISH_RESPONSE" | tee -a "$LOGFILE"
    return 1
  else
    echo "    FINAL STATUS: SUCCESS publishing '$pub_workbook_filename'." | tee -a "$LOGFILE"
    return 0
  fi
}

generate_content_diff() {
  local status="${1:0:1}"
  local current_file_path="$2"
  local previous_file_path="${3:-}"
  local file_type="${current_file_path##*.}"

  echo "---" >> "$GIT_CONTENT_DIFF_FILE"
  echo "--- Diff for: $current_file_path" >> "$GIT_CONTENT_DIFF_FILE"
  echo "---" >> "$GIT_CONTENT_DIFF_FILE"

  if [[ "$status" == "A" ]]; then
    echo "  New file added. Generating full content diff..." | tee -a "$LOGFILE"
    if [[ "$file_type" == "twb" ]]; then
      diff -u /dev/null "$current_file_path" >> "$GIT_CONTENT_DIFF_FILE" || true
    elif [[ "$file_type" == "twbx" ]]; then
      local tmp_current; tmp_current=$(mktemp -d); trap 'rm -rf "$tmp_current"' RETURN
      unzip -q "$current_file_path" -d "$tmp_current"
      local twb_current; twb_current=$(find "$tmp_current" -name "*.twb" | head -n 1)
      if [[ -f "$twb_current" ]]; then diff -u /dev/null "$twb_current" >> "$GIT_CONTENT_DIFF_FILE" || true; fi
    fi
  else
    if [[ "$file_type" == "twb" ]]; then
      git diff HEAD^1 HEAD -- "$previous_file_path" "$current_file_path" >> "$GIT_CONTENT_DIFF_FILE" || true
    elif [[ "$file_type" == "twbx" ]]; then
      echo "  TWBX modified/renamed. Extracting internal .twb for content diff..." | tee -a "$LOGFILE"
      local tmp_current; tmp_current=$(mktemp -d)
      local tmp_previous; tmp_previous=$(mktemp -d)
      trap 'rm -rf "$tmp_current" "$tmp_previous"' RETURN
      unzip -q "$current_file_path" -d "$tmp_current"
      local twb_current; twb_current=$(find "$tmp_current" -name "*.twb" | head -n 1)
      local prev_archive_path="$tmp_previous/archive.zip"; local twb_previous=""
      if git cat-file blob "HEAD^1:$previous_file_path" > "$prev_archive_path" 2>/dev/null; then
        if [[ -s "$prev_archive_path" ]]; then
          unzip -q "$prev_archive_path" -d "$tmp_previous"
          twb_previous=$(find "$tmp_previous" -name "*.twb" | head -n 1)
        fi
      else
        echo "    WARN: 'git cat-file' failed for previous version path: '$previous_file_path'. Cannot diff." | tee -a "$LOGFILE"
      fi
      if [[ -f "$twb_current" && -f "$twb_previous" ]]; then
        diff -u "$twb_previous" "$twb_current" >> "$GIT_CONTENT_DIFF_FILE" || true
        echo "    Content diff generated successfully." | tee -a "$LOGFILE"
      else
        echo "    Could not find .twb file within one or both .twbx archives for diffing." | tee -a "$LOGFILE"
      fi
    fi
  fi
  echo "" >> "$GIT_CONTENT_DIFF_FILE"
}

# ====================================================================================
# Script body
# ====================================================================================
date | tee -a "$LOGFILE"
echo "Starting Tableau Cloud Sync Process (Change-Based) with Version Validation." | tee -a "$LOGFILE"
echo "============================================================================" | tee -a "$LOGFILE"

echo "1) Authentication" | tee -a "$LOGFILE"
AUTH_RESPONSE=$(curl -s -X POST "$TABLEAU_SERVER/api/3.22/auth/signin" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d "{\\"credentials\\": {\\"name\\": \\"$TABLEAU_USER\\", \\"password\\": \\"$TABLEAU_PW\\", \\"site\\": {\\"contentUrl\\": \\"$TABLEAU_SITE_URL\\"}}}")

if echo "$AUTH_RESPONSE" | jq -e '.error' >/dev/null; then
  echo "FATAL: Authentication failed." | tee -a "$LOGFILE"
  echo "Response: $(echo "$AUTH_RESPONSE" | jq .)" | tee -a "$LOGFILE"
  exit 1
fi
TOKEN=$(echo "$AUTH_RESPONSE" | jq -r '.credentials.token')
SITE_ID=$(echo "$AUTH_RESPONSE" | jq -r '.credentials.site.id')
echo "Authentication successful." | tee -a "$LOGFILE"
echo "" | tee -a "$LOGFILE"

echo "2) Get Server Expected Workbook Version (dynamic)" | tee -a "$LOGFILE"
SERVERINFO=$(curl -s -X GET "$TABLEAU_SERVER/api/3.22/serverinfo" -H "Accept: application/json")
# Try multiple JSON paths to be robust
EXPECTED_VERSION=$(echo "$SERVERINFO" | jq -r '
  .serverInfo.productVersion.version //
  .serverInfo.productVersion //
  .serverInfo.serverVersion.version //
  .serverInfo.serverVersion //
  empty
')
if [[ -z "${EXPECTED_VERSION:-}" || "$EXPECTED_VERSION" == "null" ]]; then
  echo "WARN: Could not determine expected version from server; will proceed WITHOUT strict validation." | tee -a "$LOGFILE"
else
  echo "Server reported product version: $EXPECTED_VERSION" | tee -a "$LOGFILE"
fi
echo "" | tee -a "$LOGFILE"

echo "3) Generate Git changes from HEAD^1 vs HEAD" | tee -a "$LOGFILE"
git diff --name-status HEAD^1 HEAD > "$GIT_CHANGES_FILE"
> "$GIT_CONTENT_DIFF_FILE"

if [ ! -s "$GIT_CHANGES_FILE" ]; then
  echo "No file changes detected between HEAD^1 and HEAD. Nothing to sync." | tee -a "$LOGFILE"
  curl -s -X POST "$TABLEAU_SERVER/api/3.22/auth/signout" -H "X-Tableau-Auth: $TOKEN" > /dev/null
  exit 0
fi

echo "Found the following changes to process:" | tee -a "$LOGFILE"
cat "$GIT_CHANGES_FILE" | tee -a "$LOGFILE"
echo "" | tee -a "$LOGFILE"

# Build candidate list (only .twb/.twbx inside GIT_PROJECTS_DIR) for pre-validation
mapfile -t CANDIDATES < <(awk '{print $1" "$2" "$3}' "$GIT_CHANGES_FILE" | while read -r status p1 p2; do
  current="${p2:-$p1}"
  if [[ "$current" == "$GIT_PROJECTS_DIR/"* && ( "$current" == *.twb || "$current" == *.twbx ) ]]; then
    project_name=$(basename "$(dirname "$current")")
    if ! contains_element "$project_name" "${IGNORED_PROJECTS[@]}"; then
      echo "$status $p1 $p2"
    fi
  fi
done)

echo "4) Pre-validate workbook versions (if server version known)" | tee -a "$LOGFILE"
if [[ -n "${EXPECTED_VERSION:-}" && "$EXPECTED_VERSION" != "null" ]]; then
  VALIDATION_FAILED=0
  for triple in "${CANDIDATES[@]}"; do
    IFS=' ' read -r st p1 p2 <<< "$triple"
    current="${p2:-$p1}"
    # Skip delete-only
    if [[ "${st:0:1}" == "D" ]]; then
      continue
    fi
    if [[ -f "$current" ]]; then
      wb_ver=$(extract_workbook_version "$current" || true)
      if [[ -z "${wb_ver:-}" ]]; then
        echo "WARN: Could not detect workbook version for '$current' — skipping strict check." | tee -a "$LOGFILE"
      else
        if ver_lte "$wb_ver" "$EXPECTED_VERSION"; then
          echo "OK: '$current' version $wb_ver <= server $EXPECTED_VERSION" | tee -a "$LOGFILE"
        else
          echo "ERROR: '$current' requires version $wb_ver which is > server $EXPECTED_VERSION" | tee -a "$LOGFILE"
          VALIDATION_FAILED=1
        fi
      fi
    else
      echo "WARN: File '$current' not found locally during validation." | tee -a "$LOGFILE"
    fi
  done
  if [[ "$VALIDATION_FAILED" -ne 0 ]]; then
    echo "FATAL: Version validation failed. Aborting sync." | tee -a "$LOGFILE"
    curl -s -X POST "$TABLEAU_SERVER/api/3.22/auth/signout" -H "X-Tableau-Auth: $TOKEN" > /dev/null
    exit 1
  fi
else
  echo "Skipping strict validation (no server version available)." | tee -a "$LOGFILE"
fi
echo "" | tee -a "$LOGFILE"

echo "5) Gather current state from Tableau" | tee -a "$LOGFILE"
echo "Fetching projects..." | tee -a "$LOGFILE"
LIST_PROJECTS_RESPONSE=$(curl -s -X GET "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/projects?pageSize=1000" -H "Accept: application/json" -H "X-Tableau-Auth: $TOKEN")
declare -A tableau_projects
while IFS=$'\\t' read -r name id; do
  [[ -n "$name" ]] && tableau_projects["$name"]="$id"
done < <(echo "$LIST_PROJECTS_RESPONSE" | jq -r '.projects.project[] | "\\(.name)\\t\\(.id)"')
echo "Found ${#tableau_projects[@]} projects." | tee -a "$LOGFILE"

echo "Fetching workbooks..." | tee -a "$LOGFILE"
ALL_WBS_JSON=$(curl -s -X GET "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/workbooks?pageSize=1000" -H "Accept: application/json" -H "X-Tableau-Auth: $TOKEN")
declare -A tableau_workbooks
while IFS=$'\\t' read -r key id; do
  [[ -n "$key" ]] && tableau_workbooks["$key"]="$id"
done < <(echo "$ALL_WBS_JSON" | jq -r '.workbooks.workbook[] | "\\(.project.name)/\\(.name)\\t\\(.id)"')
echo "Found ${#tableau_workbooks[@]} workbooks." | tee -a "$LOGFILE"
echo "" | tee -a "$LOGFILE"

echo "6) Process Git changes" | tee -a "$LOGFILE"
while read -r status path1 path2; do
  current_path=${path2:-$path1}

  # Filter to repo dir + file type
  if ! [[ "$current_path" == "$GIT_PROJECTS_DIR/"* && ( "$current_path" == *.twb || "$current_path" == *.twbx ) ]]; then
    echo "SKIP: Change to '$current_path' is outside '$GIT_PROJECTS_DIR' or not a .twb/.twbx. Ignoring." | tee -a "$LOGFILE"
    continue
  fi

  project_name=$(basename "$(dirname "$current_path")")
  if contains_element "$project_name" "${IGNORED_PROJECTS[@]}"; then
    echo "SKIP: Project '$project_name' is in IGNORED_PROJECTS. Ignoring '$(basename "$current_path")'." | tee -a "$LOGFILE"
    continue
  fi

  # Per-file guard (in case commit was pushed between validation & publish)
  if [[ -n "${EXPECTED_VERSION:-}" && "$EXPECTED_VERSION" != "null" && -f "$current_path" && "${status:0:1}" != "D" ]]; then
    this_ver=$(extract_workbook_version "$current_path" || true)
    if [[ -n "${this_ver:-}" && ! $(ver_lte "$this_ver" "$EXPECTED_VERSION") ]]; then
      echo "ERROR: '$current_path' version $this_ver > server $EXPECTED_VERSION. Skipping publish." | tee -a "$LOGFILE"
      continue
    fi
  fi

  case ${status:0:1} in
    A)
      echo "PROCESS (ADD): '$current_path'" | tee -a "$LOGFILE"
      if publish_workbook "$current_path"; then
        generate_content_diff "A" "$current_path"
      fi
      ;;
    M)
      echo "PROCESS (MODIFY): '$current_path'" | tee -a "$LOGFILE"
      if publish_workbook "$current_path"; then
        generate_content_diff "M" "$current_path" "$current_path"
      fi
      ;;
    D)
      echo "PROCESS (DELETE): '$path1'" | tee -a "$LOGFILE"
      del_project_name=$(basename "$(dirname "$path1")")
      del_workbook_name_no_ext="${path1##*/}"; del_workbook_name_no_ext="${del_workbook_name_no_ext%.*}"
      wb_key="$del_project_name/$del_workbook_name_no_ext"
      if [[ -n "${tableau_workbooks[$wb_key]+_}" ]]; then
        wb_id=${tableau_workbooks[$wb_key]}
        echo "  Deleting workbook '$wb_key' (id=$wb_id)..." | tee -a "$LOGFILE"
        curl -s -X DELETE "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/workbooks/$wb_id" -H "X-Tableau-Auth: $TOKEN" >/dev/null
        echo "    FINAL STATUS: Deletion request sent." | tee -a "$LOGFILE"
      else
        echo "  WARN: Workbook '$wb_key' marked for delete not found in Tableau." | tee -a "$LOGFILE"
      fi
      ;;
    R)
      echo "PROCESS (RENAME): '$path1' -> '$path2'" | tee -a "$LOGFILE"
      echo "  RENAME (DELETE old):" | tee -a "$LOGFILE"
      del_project_name=$(basename "$(dirname "$path1")")
      old_workbook_name_no_ext="${path1##*/}"; old_workbook_name_no_ext="${old_workbook_name_no_ext%.*}"
      old_wb_key="$del_project_name/$old_workbook_name_no_ext"
      if [[ -n "${tableau_workbooks[$old_wb_key]+_}" ]]; then
        wb_id=${tableau_workbooks[$old_wb_key]}
        echo "    Deleting old workbook '$old_wb_key' (id=$wb_id)..." | tee -a "$LOGFILE"
        curl -s -X DELETE "$TABLEAU_SERVER/api/3.22/sites/$SITE_ID/workbooks/$wb_id" -H "X-Tableau-Auth: $TOKEN" >/dev/null
        echo "    Deletion request sent." | tee -a "$LOGFILE"
      else
        echo "    WARN: Old workbook '$old_wb_key' not found in Tableau." | tee -a "$LOGFILE"
      fi
      echo "  RENAME (PUBLISH new):" | tee -a "$LOGFILE"
      if publish_workbook "$path2"; then
        generate_content_diff "R" "$path2" "$path1"
      fi
      ;;
    *)
      echo "WARN: Unknown git status '$status' for '$path1'. Ignoring." | tee -a "$LOGFILE"
      ;;
  esac
  echo "" | tee -a "$LOGFILE"
done < "$GIT_CHANGES_FILE"

echo "7) Sign out" | tee -a "$LOGFILE"
curl -s -X POST "$TABLEAU_SERVER/api/3.22/auth/signout" -H "X-Tableau-Auth: $TOKEN" >/dev/null
echo "Session token invalidated." | tee -a "$LOGFILE"
echo "" | tee -a "$LOGFILE"
echo "===================================" | tee -a "$LOGFILE"
echo "Sync Process Completed." | tee -a "$LOGFILE"
date | tee -a "$LOGFILE"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'log.txt,git_file_changes.txt,git_filetext_changes.txt', fingerprint: true
        }
        failure {
            mail to: 'team@example.com',
                 subject: "Jenkins Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Something went wrong in Tableau sync. Check Jenkins logs."
        }
    }
}
