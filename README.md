name: Update LeetCode Stats

on:
  schedule:
    - cron: "0 */6 * * *"

  workflow_dispatch:

permissions:
  contents: write

jobs:
  update-leetcode:
    runs-on: ubuntu-latest

    steps:

      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Fetch LeetCode Stats
        run: |

          python - <<'PY'

          import json
          import urllib.request
          import sys

          USERNAME = "enum101"

          query = """
          query getUserProfile($username: String!) {
            matchedUser(username: $username) {
              username

              submitStats: submitStatsGlobal {
                acSubmissionNum {
                  difficulty
                  count
                }
              }
            }
          }
          """

          payload = json.dumps({
              "query": query,
              "variables": {
                  "username": USERNAME
              }
          }).encode("utf-8")

          request = urllib.request.Request(
              "https://leetcode.com/graphql",
              data=payload,
              headers={
                  "Content-Type": "application/json",
                  "Referer": "https://leetcode.com/",
                  "User-Agent": "Mozilla/5.0"
              },
              method="POST"
          )

          try:

              with urllib.request.urlopen(request) as response:
                  data = json.loads(
                      response.read().decode("utf-8")
                  )

          except Exception as e:

              print("Failed to connect to LeetCode")
              print(e)
              sys.exit(1)

          if "errors" in data:

              print("LeetCode returned an error:")
              print(data["errors"])
              sys.exit(1)

          user = data["data"]["matchedUser"]

          if user is None:

              print("LeetCode username not found:")
              print(USERNAME)
              sys.exit(1)

          stats = user["submitStats"]["acSubmissionNum"]

          solved = {}

          for item in stats:

              difficulty = item["difficulty"]
              count = item["count"]

              solved[difficulty] = count

          total = solved.get("All", 0)
          easy = solved.get("Easy", 0)
          medium = solved.get("Medium", 0)
          hard = solved.get("Hard", 0)

          print("================================")
          print("       LEETCODE STATISTICS")
          print("================================")

          print(f"Total  : {total}")
          print(f"Easy   : {easy}")
          print(f"Medium : {medium}")
          print(f"Hard   : {hard}")

          print("================================")

          with open("leetcode_stats.txt", "w") as file:

              file.write(
                  f"{total}|{easy}|{medium}|{hard}"
              )

          PY


      - name: Update README
        run: |

          python - <<'PY'

          from datetime import datetime, timezone

          with open("leetcode_stats.txt", "r") as file:

              total, easy, medium, hard = (
                  file.read()
                  .strip()
                  .split("|")
              )

          today = datetime.now(
              timezone.utc
          ).strftime("%d %B %Y")


          new_section = f"""<!-- LEETCODE-STATS:START -->

          <div align="center">

          <a href="https://leetcode.com/u/enum101/">

          <img src="https://leetcard.jacoblin.cool/enum101?theme=dark&font=baloo&ext=heatmap" alt="LeetCode Stats">

          </a>

          <br><br>

          | 🧩 Total Solved | 🟢 Easy | 🟡 Medium | 🔴 Hard |
          |:---:|:---:|:---:|:---:|
          | **{total}** | **{easy}** | **{medium}** | **{hard}** |

          <br>

          <sub>Last updated: {today}</sub>

          </div>

          <!-- LEETCODE-STATS:END -->"""


          with open(
              "README.md",
              "r",
              encoding="utf-8"
          ) as file:

              readme = file.read()


          start = "<!-- LEETCODE-STATS:START -->"
          end = "<!-- LEETCODE-STATS:END -->"


          if start in readme and end in readme:

              before = readme.split(start)[0]

              after = readme.split(end)[1]

              readme = (
                  before
                  + new_section
                  + after
              )

          else:

              readme += (
                  "\n\n"
                  + new_section
                  + "\n"
              )


          with open(
              "README.md",
              "w",
              encoding="utf-8"
          ) as file:

              file.write(readme)

          PY


      - name: Commit changes
        run: |

          git config user.name "github-actions[bot]"

          git config user.email \
            "41898282+github-actions[bot]@users.noreply.github.com"

          git add README.md

          if git diff --cached --quiet; then

            echo "No changes to commit"

          else

            git commit -m "Update LeetCode stats"

            git push

          fi
