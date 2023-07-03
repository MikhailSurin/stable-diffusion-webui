Мои расследования - как сохранить пути для каждого пользователя
===============================================================

1. Пути хранятся в файле config.json
2. нужно найти в коде строку - outdir_txt2img_samples
3. и сделать замену подстроки %user% на имя текущего залогиненого пользователя
4. тогда картинки будут сохраняться в именной каталог

Как получить имя текущего пользователя?

venv/lib/python3.11/site-packages/gradio/routes.py:165

        @app.get("/user")
        @app.get("/user/")
        def get_current_user(request: fastapi.Request) -> Optional[str]:
            token = request.cookies.get("access-token") or request.cookies.get(
                "access-token-unsecure"
            )
            return app.tokens.get(token)



        @app.post("/login")
        @app.post("/login/")
        def login(form_data: OAuth2PasswordRequestForm = Depends()):
            username, password = form_data.username, form_data.password
            if app.auth is None:
                return RedirectResponse(url="/", status_code=status.HTTP_302_FOUND)
            if (
                not callable(app.auth)
                and username in app.auth
                and app.auth[username] == password
            ) or (callable(app.auth) and app.auth.__call__(username, password)):
                token = secrets.token_urlsafe(16)
                app.tokens[token] = username
                response = JSONResponse(content={"success": True})
                response.set_cookie(
                    key="access-token",
                    value=token,
                    httponly=True,
                    samesite="none",
                    secure=True,
                )
                response.set_cookie(
                    key="access-token-unsecure", value=token, httponly=True
                )
                return response
            else:
                raise HTTPException(status_code=400, detail="Incorrect credentials.")


modules/images.py:583

os.makedirs
