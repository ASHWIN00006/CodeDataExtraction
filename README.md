# CodeDataExtraction
!pip install asyncpraw

# Importing the necessary libraries
import nest_asyncio
import asyncio
import asyncpraw #since GoogleColab is an Asynchronous Environment
import pandas as pd
from datetime import datetime
from google.colab import userdata

# Allows asynchronous code to run in Google Colab
nest_asyncio.apply()

# A global Reddit object we'll initialize later
reddit = None

async def authenticate_with_reddit():
    """
    🔐 Authenticate using credentials stored in Google Colab's `userdata`.
    These should include your Reddit API client_id, client_secret, and user_agent.
    """
    global reddit
    reddit = asyncpraw.Reddit(
        client_id=userdata.get("client_id"),
        client_secret=userdata.get("client_secret"),
        user_agent=userdata.get("user_agent")
    )
    print("✅ Successfully authenticated with Reddit!")
    return reddit

#  Define the date range for which we want to collect data
start_timestamp = datetime(2025, 1, 1).timestamp()
end_timestamp = datetime(2025, 4, 15).timestamp()

async def collect_subreddit_data(subreddit_name):
    """
    Collect posts and their comments from a given subreddit within the defined date range.
    """
    global reddit
    if reddit is None:
        await authenticate_with_reddit()

    print(f"Starting data collection from r/{subreddit_name} (Jan 1 – Apr 15, 2025)...")
    collected_data = []

    try:
        subreddit = await reddit.subreddit(subreddit_name)

        # 🔁 Go through up to 1000 new posts
        async for post in subreddit.new(limit=1000):
            if not (start_timestamp <= post.created_utc <= end_timestamp):
                continue  # ⏩ Skip posts outside our date range

            try:
                await post.load()
            except Exception:
                continue  # Skip if the post fails to load

            # 📝 Save post (submission) info as a data row
            collected_data.append({
                "subreddit": subreddit_name,
                "submission_id": post.id,
                "parent_id": post.id,
                "target_author": "",
                "comment_id": post.id,
                "source_author": post.author.name if post.author else "[deleted]",
                "created_utc": datetime.utcfromtimestamp(post.created_utc).strftime('%Y-%m-%d %H:%M:%S'),
                "comment_score": "",
                "post_score": post.score,
                "text": post.selftext if post.selftext else "",
                "title": post.title
            })

            # 💬 Load and iterate through all comments
            try:
                await post.comments.replace_more(limit=0)
            except Exception:
                continue  # Skip if comments can't be loaded

            comments = post.comments.list() if hasattr(post.comments, 'list') else []

            for comment in comments:
                if not hasattr(comment, "body") or comment.body is None:
                    continue

                # 🔄 Determine who the comment is replying to
                if comment.parent_id.startswith("t3_"):
                    # Top-level comment → replying to the post
                    target_author = post.author.name if post.author else "[deleted]"
                else:
                    # Nested comment → replying to another comment
                    try:
                        parent = await reddit.comment(comment.parent_id.split("_")[1])
                        target_author = parent.author.name if parent.author else "[deleted]"
                    except Exception:
                        target_author = "[parent comment deleted]"

                # 💾 Save comment data
                collected_data.append({
                    "subreddit": subreddit_name,
                    "submission_id": post.id,
                    "parent_id": comment.parent_id.split("_")[1],
                    "target_author": target_author,
                    "comment_id": comment.id,
                    "source_author": comment.author.name if comment.author else "[deleted]",
                    "created_utc": datetime.utcfromtimestamp(comment.created_utc).strftime('%Y-%m-%d %H:%M:%S'),
                    "comment_score": comment.score,
                    "post_score": post.score,
                    "text": comment.body,
                    "title": post.title
                })

            await asyncio.sleep(2)  # 💤 Respect API rate limits

    except Exception as e:
        print(f"❌ Something went wrong while fetching data from r/{subreddit_name}: {e}")
        import traceback
        traceback.print_exc()

    await save_to_csv(collected_data, subreddit_name)

async def save_to_csv(data, subreddit_name):
    """
    💾 Save the collected data into a CSV file.
    """
    if not data:
        print(f"⚠️ No data found for r/{subreddit_name}")
        return

    df = pd.DataFrame(data)
    filename = f"{subreddit_name}_JanApr2025_cleaned.csv"
    df.to_csv(filename, index=False)
    print(f"✅ Data successfully saved to: {filename}")

async def main():
    # 🧠 You can change the subreddit name here
    await collect_subreddit_data("roosterteeth")

    if reddit:
        await reddit.close()

# 🚀 Run everything
if __name__ == "__main__":
    asyncio.run(main())
