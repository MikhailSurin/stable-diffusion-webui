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



        @app.get("/token")
        @app.get("/token/")
        def get_token(request: fastapi.Request) -> dict:
            token = request.cookies.get("access-token")
            return {"token": token, "user": app.tokens.get(token)}


modules/images.py:583

os.makedirs


setup_middleware(app)


modules/shared.py:708

opts = Options()
if os.path.exists(config_filename):
    opts.load(config_filename)


1)
extensions/stable-diffusion-webui-images-browser/scripts/image_browser.py:103
path_maps = {
    "txt2img": opts.outdir_samples or opts.outdir_txt2img_samples,
    "img2img": opts.outdir_samples or opts.outdir_img2img_samples,
    "txt2img-grids": opts.outdir_grids or opts.outdir_txt2img_grids,
    "img2img-grids": opts.outdir_grids or opts.outdir_img2img_grids,
    "Extras": opts.outdir_samples or opts.outdir_extras_samples,
    favorite_tab_name: opts.outdir_save
}

2)
modules/txt2img.py:15
    p = processing.StableDiffusionProcessingTxt2Img(
        sd_model=shared.sd_model,
        outpath_samples=opts.outdir_samples or opts.outdir_txt2img_samples,
        outpath_grids=opts.outdir_grids or opts.outdir_txt2img_grids,


3)
modules/api/api.py:330
        with self.queue_lock:
            p = StableDiffusionProcessingTxt2Img(sd_model=shared.sd_model, **args)
            p.scripts = script_runner
            p.outpath_grids = opts.outdir_txt2img_grids
            p.outpath_samples = opts.outdir_txt2img_samples

4)
modules/ui.py:525
            txt2img_gallery, generation_info, html_info, html_log = create_output_panel("txt2img", opts.outdir_txt2img_samples)
