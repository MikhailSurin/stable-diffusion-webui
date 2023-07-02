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


modules/images.py:583

os.makedirs
