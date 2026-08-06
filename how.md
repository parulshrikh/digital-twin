cp .env.example .env
docker compose up -d
pip install -r requirements.txt
python scripts/apply_schema.py
python scripts/seed_dev_data.py
uvicorn app.main:app --reload