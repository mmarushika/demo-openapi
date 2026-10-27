pipeline {
    agent any

    options {
        timestamps()
    }

    stages {
        stage('Compatibility Check') {
            when {
                changeRequest()
            }
            options {
                timeout(time: 6, unit: 'MINUTES')
            }
            steps {
                sh '''
                    set -eu
                    RESULT=FAIL
                    trap 'printf "%s\\n" "Compatibility check completed"; if [ "${RESULT}" = PASS ]; then printf "%s\\n" "PASS: Compatibility check passed."; else printf "%s\\n" "FAIL: Compatibility check failed."; fi' 0

                    fail() {
                        printf 'FAIL: %s\\n' "$1" >&2
                        exit 1
                    }

                    print_changes() {
                        CHANGE_COUNT=$(jq 'if (.changes|type) == "array" then (.changes|length) else 0 end' \
                            response.json)
                        jq -c 'if (.changes|type) == "array" then .changes[:20][] else empty end' \
                            response.json
                        if [ "${CHANGE_COUNT}" -gt 20 ]; then
                            printf 'Omitted %s additional change(s).\\n' "$((CHANGE_COUNT - 20))"
                        fi
                    }

                    # FIX: remove leftovers from earlier builds so stale reports are never shown or archived
                    rm -f old.yml new.yml body.json response.json

                    printf '%s\\n' "Compatibility check started"
                    printf 'PR: %s | Source branch: %s | Target branch: %s\\n' \
                        "${CHANGE_ID:-unknown}" "${CHANGE_BRANCH:-unknown}" "${CHANGE_TARGET:-unknown}"

                    command -v curl >/dev/null 2>&1 || fail "curl is required on the Jenkins agent."
                    command -v jq >/dev/null 2>&1 || fail "jq is required on the Jenkins agent."

                    [ -f openapi.yml ] || fail "openapi.yml is missing from the checked-out PR workspace."
                    [ -s openapi.yml ] || fail "openapi.yml in the checked-out PR workspace is empty."

                    # FIX: a plain fetch only sets FETCH_HEAD (no origin/<branch> ref exists in this checkout)
                    git fetch --no-tags origin "${CHANGE_TARGET}" ||
                        fail "Could not fetch target branch '${CHANGE_TARGET}'."
                    git show FETCH_HEAD:openapi.yml > old.yml ||
                        fail "openapi.yml is missing from target branch '${CHANGE_TARGET}'."
                    [ -s old.yml ] ||
                        fail "openapi.yml on target branch '${CHANGE_TARGET}' is empty."

                    cp openapi.yml new.yml ||
                        fail "Could not copy openapi.yml from the checked-out PR workspace."
                    [ -s new.yml ] ||
                        fail "openapi.yml in the checked-out PR workspace is empty."

                    jq -n --rawfile old old.yml --rawfile new new.yml \
                        '{oldDocument:$old, newDocument:$new}' > body.json ||
                        fail "Could not build the comparison request."

                    curl -s -o /dev/null --max-time 90 https://api-compatibility.onrender.com/ || true

                    HTTP_CODE=''
                    if HTTP_CODE=$(curl -s -S -o response.json -w '%{http_code}' --max-time 120 \
                        -X POST -H 'Content-Type: application/json' --data-binary @body.json \
                        'https://api-compatibility.onrender.com/api/compare/report?failOnBreaking=false'); then
                        CURL_EXIT=0
                    else
                        CURL_EXIT=$?
                    fi

                    if [ "${CURL_EXIT}" -ne 0 ]; then
                        RESPONSE_PREVIEW=$(head -c 500 response.json 2>/dev/null || true)
                        printf 'FAIL: API request failed (curl exit %s, HTTP status %s). Response: %s\\n' \
                            "${CURL_EXIT}" "${HTTP_CODE:-000}" "${RESPONSE_PREVIEW}" >&2
                        exit 1
                    fi

                    case "${HTTP_CODE}" in
                        2[0-9][0-9]) ;;
                        *)
                            RESPONSE_PREVIEW=$(head -c 500 response.json 2>/dev/null || true)
                            printf 'FAIL: API returned HTTP status %s. Response: %s\\n' \
                                "${HTTP_CODE:-000}" "${RESPONSE_PREVIEW}" >&2
                            exit 1
                            ;;
                    esac

                    if ! jq -e . response.json >/dev/null 2>&1; then
                        fail "API response is not valid JSON."
                    fi

                    SUMMARY=$(jq -c '.summary // empty' response.json) ||
                        fail "Could not read the API response summary."
                    if [ -n "${SUMMARY}" ]; then
                        printf 'Summary: %s\\n' "${SUMMARY}"
                    fi

                    if ! jq -e 'has("breaking") and (.breaking|type=="boolean")' \
                        response.json >/dev/null; then
                        print_changes
                        fail "API response is missing a boolean 'breaking' field."
                    fi

                    if jq -e '.breaking == false' response.json >/dev/null; then
                        RESULT=PASS
                    else
                        printf '%s\\n' "FAIL: Breaking changes detected." >&2
                        print_changes
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'response.json', allowEmptyArchive: true
            // FIX: also remove response.json after it has been archived
            sh 'rm -f old.yml new.yml body.json response.json'
        }
    }
}