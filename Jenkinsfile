from flask import Flask, request, Response
import os, zipfile, difflib, tempfile, requests, threading, time, traceback
from dotenv import load_dotenv

load_dotenv()
app = Flask(__name__)

GITLAB_TOKEN = os.getenv("GITLAB_TOKEN")
GITLAB_API = os.getenv("GITLAB_API")
BOT_USERNAME = os.getenv("BOT_USERNAME", "tableau-diff-bot")
MAX_LINES_PER_COMMENT = 1000


def extract_twb_content(path, original_name):
    if original_name.endswith(".twb"):
        try:
            with open(path, "r", encoding="utf-8") as f:
                return f.read()
        except Exception as e:
            print(f"[extract] Error reading .twb: {e}")
            traceback.print_exc()
    elif original_name.endswith(".twbx"):
        try:
            with zipfile.ZipFile(path, 'r') as z:
                twb_files = [f for f in z.namelist() if f.endswith(".twb")]
                if not twb_files:
                    print(f"[extract] No .twb file found inside {path}")
                    return ""
                with z.open(twb_files[0]) as twb_file:
                    return twb_file.read().decode("utf-8")
        except Exception as e:
            print(f"[extract] Error with .twbx: {e}")
            traceback.print_exc()
    return ""


def generate_minimal_diff(old_content, new_content):
    diff = difflib.unified_diff(
        old_content.splitlines(),
        new_content.splitlines(),
        fromfile='old.twb',
        tofile='new.twb',
        lineterm=''
    )
    return [line for line in diff if line.startswith(('+', '-', '@@'))]


def post_gitlab_comment(project_id, mr_iid, body):
    headers = {"PRIVATE-TOKEN": GITLAB_TOKEN}
    note_url = f"{GITLAB_API}/projects/{project_id}/merge_requests/{mr_iid}/notes"
    response = requests.post(note_url, headers=headers, json={"body": body})
    print(f"📩 Posted comment: {response.status_code}")


def paginate_and_post_comment(project_id, mr_iid, file_path, content, label="xml"):
    lines = content.splitlines()
    for i in range(0, len(lines), MAX_LINES_PER_COMMENT):
        chunk = lines[i:i + MAX_LINES_PER_COMMENT]
        part = i // MAX_LINES_PER_COMMENT + 1
        total = (len(lines) - 1) // MAX_LINES_PER_COMMENT + 1
        comment = (
            f"📄 `{file_path}` (Part {part}/{total})\n\n"
            f"```{label}\n" + "\n".join(chunk) + "\n```"
        )
        post_gitlab_comment(project_id, mr_iid, comment)


def process_webhook(payload):
    try:
        if payload.get("object_kind") != "merge_request":
            print("⚠️ Not a merge request. Ignored.")
            return

        mr = payload["object_attributes"]
        project = payload["project"]
        project_id = project["id"]
        mr_iid = mr["iid"]

        headers = {"PRIVATE-TOKEN": GITLAB_TOKEN}
        changes_url = f"{GITLAB_API}/projects/{project_id}/merge_requests/{mr_iid}/changes"
        changes = requests.get(changes_url, headers=headers).json()

        for change in changes.get("changes", []):
            file_path = change["new_path"]
            if not (file_path.endswith(".twb") or file_path.endswith(".twbx")):
                continue

            is_new = change.get("new_file", False)
            is_deleted = change.get("deleted_file", False)

            old_url = f"{GITLAB_API}/projects/{project_id}/repository/files/{change['old_path'].replace('/', '%2F')}/raw?ref={mr['target_branch']}"
            new_url = f"{GITLAB_API}/projects/{project_id}/repository/files/{change['new_path'].replace('/', '%2F')}/raw?ref={mr['source_branch']}"

            with tempfile.TemporaryDirectory() as tmpdir:
                old_path = os.path.join(tmpdir, "old_file")
                new_path = os.path.join(tmpdir, "new_file")
                old_xml = ""
                new_xml = ""

                if not is_new:
                    for _ in range(3):
                        r_old = requests.get(old_url, headers=headers)
                        if r_old.status_code == 200 and len(r_old.content) > 1000:
                            with open(old_path, 'wb') as f:
                                f.write(r_old.content)
                                f.flush()
                                os.fsync(f.fileno())
                            time.sleep(0.2)
                            old_xml = extract_twb_content(old_path, change['old_path'])
                            break

                if not is_deleted:
                    for _ in range(3):
                        r_new = requests.get(new_url, headers=headers)
                        if r_new.status_code == 200 and len(r_new.content) > 1000:
                            with open(new_path, 'wb') as f:
                                f.write(r_new.content)
                                f.flush()
                                os.fsync(f.fileno())
                            time.sleep(0.2)
                            new_xml = extract_twb_content(new_path, change['new_path'])
                            break

                # New File
                if is_new and new_xml:
                    paginate_and_post_comment(project_id, mr_iid, file_path, new_xml, label="xml")
                    continue

                # Deleted File
                if is_deleted and old_xml:
                    paginate_and_post_comment(project_id, mr_iid, file_path, old_xml, label="xml")
                    continue

                # Modified File
                if old_xml and new_xml:
                    diff_lines = generate_minimal_diff(old_xml, new_xml)
                    if not diff_lines:
                        post_gitlab_comment(project_id, mr_iid, f"✅ No changes in `{file_path}`")
                        continue

                    for i in range(0, len(diff_lines), MAX_LINES_PER_COMMENT):
                        chunk = diff_lines[i:i + MAX_LINES_PER_COMMENT]
                        part = i // MAX_LINES_PER_COMMENT + 1
                        total = (len(diff_lines) - 1) // MAX_LINES_PER_COMMENT + 1
                        comment = (
                            f"🔁 Diff for `{file_path}` (Part {part}/{total})\n\n"
                            f"```diff\n" + "\n".join(chunk) + "\n```"
                        )
                        post_gitlab_comment(project_id, mr_iid, comment)
                else:
                    post_gitlab_comment(project_id, mr_iid, f"⚠️ Could not extract content from `{file_path}`")

    except Exception as e:
        print(f"❌ Error in process_webhook: {e}")
        traceback.print_exc()


@app.route("/webhook", methods=["POST"])
def handle_webhook():
    try:
        payload = request.json
        print("📥 Webhook received")
        threading.Thread(target=process_webhook, args=(payload,)).start()
        return Response(status=200)
    except Exception as e:
        print(f"❌ Webhook error: {e}")
        traceback.print_exc()
        return Response(status=500)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
