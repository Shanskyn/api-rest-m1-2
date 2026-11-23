# TP3 : Le squelette 
> Dorian BORDERAN | ESGI M1 SI 2026-2027

```bash
(.venv) debian@debian1:~/api-rest-m1-1/adminapi$ pip install fastapi uvicorn python-dotenv
Collecting fastapi
  Downloading fastapi-0.162.0-py3-none-any.whl.metadata (26 kB)
Collecting uvicorn
  Downloading uvicorn-0.54.0-py3-none-any.whl.metadata (6.6 kB)
Collecting python-dotenv
  Downloading python_dotenv-1.2.4-py3-none-any.whl.metadata (29 kB)
Collecting starlette>=0.46.0 (from fastapi)
  Downloading starlette-1.10.0-py3-none-any.whl.metadata (6.6 kB)
Collecting pydantic>=2.9.0 (from fastapi)
  Downloading pydantic-2.14.1-py3-none-any.whl.metadata (133 kB)
Collecting typing-extensions>=4.8.0 (from fastapi)
  Downloading typing_extensions-4.16.0-py3-none-any.whl.metadata (3.3 kB)
Collecting typing-inspection>=0.4.2 (from fastapi)
  Downloading typing_inspection-0.4.4-py3-none-any.whl.metadata (2.6 kB)
Collecting annotated-doc>=0.0.2 (from fastapi)
  Downloading annotated_doc-0.0.5-py3-none-any.whl.metadata (6.5 kB)
Collecting opentelemetry-api>=1.44.0 (from fastapi)
  Downloading opentelemetry_api-1.45.1-py3-none-any.whl.metadata (1.4 kB)
Collecting click>=7.0 (from uvicorn)
  Downloading click-8.5.0-py3-none-any.whl.metadata (2.6 kB)
Collecting h11>=0.8 (from uvicorn)
  Downloading h11-0.16.0-py3-none-any.whl.metadata (8.3 kB)
Collecting annotated-types>=0.6.0 (from pydantic>=2.9.0->fastapi)
  Downloading annotated_types-0.8.0-py3-none-any.whl.metadata (15 kB)
Collecting pydantic-core==2.50.1 (from pydantic>=2.9.0->fastapi)
  Downloading pydantic_core-2.50.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl.metadata (6.5 kB)
Collecting anyio<5,>=4.0.0 (from starlette>=0.46.0->fastapi)
  Downloading anyio-4.15.1-py3-none-any.whl.metadata (4.7 kB)
Requirement already satisfied: idna>=2.8 in /home/debian/api-rest-m1-1/.venv/lib/python3.13/site-packages (from anyio<5,>=4.0.0->starlette>=0.46.0->fastapi) (3.20)
Downloading fastapi-0.162.0-py3-none-any.whl (148 kB)
Downloading uvicorn-0.54.0-py3-none-any.whl (87 kB)
Downloading python_dotenv-1.2.4-py3-none-any.whl (23 kB)
Downloading annotated_doc-0.0.5-py3-none-any.whl (5.3 kB)
Downloading click-8.5.0-py3-none-any.whl (125 kB)
Downloading h11-0.16.0-py3-none-any.whl (37 kB)
Downloading opentelemetry_api-1.45.1-py3-none-any.whl (60 kB)
Downloading pydantic-2.14.1-py3-none-any.whl (488 kB)
Downloading pydantic_core-2.50.1-cp313-cp313-manylinux_2_17_x86_64.manylinux2014_x86_64.whl (2.2 MB)
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 2.2/2.2 MB 8.8 MB/s eta 0:00:00
Downloading annotated_types-0.8.0-py3-none-any.whl (13 kB)
Downloading starlette-1.10.0-py3-none-any.whl (80 kB)
Downloading anyio-4.15.1-py3-none-any.whl (132 kB)
Downloading typing_extensions-4.16.0-py3-none-any.whl (45 kB)
Downloading typing_inspection-0.4.4-py3-none-any.whl (14 kB)
Installing collected packages: typing-extensions, python-dotenv, h11, click, annotated-types, annotated-doc, uvicorn, typing-inspection, pydantic-core, opentelemetry-api, anyio, starlette, pydantic, fastapi
Successfully installed annotated-doc-0.0.5 annotated-types-0.8.0 anyio-4.15.1 click-8.5.0 fastapi-0.162.0 h11-0.16.0 opentelemetry-api-1.45.1 pydantic-2.14.1 pydantic-core-2.50.1 python-dotenv-1.2.4 starlette-1.10.0 typing-extensions-4.16.0 typing-inspection-0.4.4 uvicorn-0.54.0
(.venv) debian@debian1:~/api-rest-m1-1/adminapi$ pip freeze > requirements.txt
(.venv) debian@debian1:~/api-rest-m1-1/adminapi$ cd ..
(.venv) debian@debian1:~/api-rest-m1-1$ pip freeze > requirements.txt
```

```bash
(.venv) debian@debian1:~/api-rest-m1-1/adminapi$ uvicorn app.main:app --reload --env-file .env --port 8000
INFO:     Will watch for changes in these directories: ['/home/debian/api-rest-m1-1/adminapi']
INFO:     Loading environment from '.env'
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [42215] using StatReload
INFO:     Started server process [42217]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     127.0.0.1:54730 - "GET /docs HTTP/1.1" 200 OK
INFO:     127.0.0.1:54730 - "GET /openapi.json HTTP/1.1" 200 OK
```

```bash
debian@debian1 ~/a/adminapi (main) [127]> python3 verifier.py lecture


== lecture : 7/7 ==
  [ok] GET /health répond 200 et {"status": "ok"}
  [ok] GET /containers répond 200 avec une liste JSON
  [ok] la liste contient le conteneur étiqueté, avec id, name, image et state
  [ok] la liste ne montre pas les conteneurs sans label
  [ok] GET /containers/{id} répond 200 avec le bon conteneur
  [ok] GET /containers/{id} répond 404 pour un identifiant inconnu
  [ok] GET /containers/{id} répond 404 pour un conteneur sans label

Total : 7/7 vérifications passées.
```