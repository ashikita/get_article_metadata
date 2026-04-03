ChatGPTへの質問

Crossref APIを使って論文の著者情報を取得するPythonコードを教えてください。
その際、クエリに「mailto=qir@m.kyushu-u.ac.jp」パラメータを含めるか、
または、User-Agentヘッダーに「mailto: qir@m.kyushu-u.ac.jp」を含めます。
また、1秒間に50回のリクエストレートとしてください。
処理が完了したら標準出力に完了メッセージと処理件数と処理時間を出力してください。

処理手順は以下の通りです。

1．DOIのリストを"doi_list.txt"から取得します。

2. DOIから以下の著者情報を取得します。
・"ORCID": "string" (http://...となっていたらhttps://...に変換)
・"family": "string" 及び "given": "string" を", "で結合
・"affiliation": [{"name": "string"}]の"string"を複数ある場合は", "で結合
・"sequence": "string"

3. 取得した著者情報は以下の構成で"authors_metadata.tsv"に出力します。
"DOI queue number"\t "DOI" \t "author queue number" \t "ORCID" \t "given, family" \t "affiliation" \t "sequence"

なお、Crossref APIで取得する著者情報のデータ構造は以下の通りです。

{
  "message": {
    "author": [
      {
        "ORCID": "string",
        "suffix": "string",
        "given": "string",
        "family": "string",
        "affiliation": [{"name": "string"}]
        "name": "string",
        "prefix": "string",
        "sequence": "string"
      }
    ],
   ],
  }
}


========

ChatGPTの回答
import requests
import json
import time

API_URL = "https://api.crossref.org/works/"
RATE_LIMIT = 50
EMAIL_PARAM = "mailto:qir@m.kyushu-u.ac.jp"
DOIs_file = "doi_list.txt"
OUTPUT_FILE = "authors_metadata.tsv"

def fetch_author_info(doi):
    headers = {
        "User-Agent": EMAIL_PARAM
    }

    url = f"{API_URL}{doi}"
    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        data = response.json()
        authors = data.get("message", {}).get("author", [])

        author_info = []
        for i, author in enumerate(authors, start=1):
            orcid = author.get("ORCID", "")
            given_name = author.get("given", "")
            family_name = author.get("family", "")
            affiliation = ", ".join([aff.get("name", "") for aff in author.get("affiliation", [])])
            sequence = author.get("sequence", "")

            # Convert ORCID URLs from http to https if present
            orcid = orcid.replace("http://", "https://")

            author_info.append((orcid, f"{family_name}, {given_name}", affiliation, sequence))

        return author_info
    else:
        print(f"Failed to fetch data for DOI: {doi}, Status code: {response.status_code}")
        return []

def main():
    with open(DOIs_file, "r") as f:
        dois = f.read().splitlines()

    with open(OUTPUT_FILE, "w") as output_file:
        output_file.write("DOI queue number\tDOI\tAuthor queue number\tORCID\tName\tAffiliation\tSequence\n")

        total_processed = 0
        start_time = time.time()

        for i, doi in enumerate(dois, start=1):
            author_info = fetch_author_info(doi)

            for j, (orcid, name, affiliation, sequence) in enumerate(author_info, start=1):
                output_file.write(f"{i}\t{doi}\t{j}\t{orcid}\t{name}\t{affiliation}\t{sequence}\n")

            total_processed += len(author_info)
            time_elapsed = time.time() - start_time

            if total_processed % RATE_LIMIT == 0 and i < len(dois):
                time.sleep(1)

        print(f"Processing completed. Processed {total_processed} entries in {time_elapsed:.2f} seconds.")

if __name__ == "__main__":
    main()
