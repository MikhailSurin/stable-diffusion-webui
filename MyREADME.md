Мои расследования - как сохранить пути для каждого пользователя
===============================================================

http://0.0.0.0:8000/docs

http://0.0.0.0:8000/redoc

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


        @app.websocket("/queue/join")
        async def join_queue(
            websocket: WebSocket,
            token: Optional[str] = Depends(ws_login_check),
        ):
            blocks = app.get_blocks()
            if app.auth is not None and token is None:
                await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
                return
            if blocks._queue.server_path is None:
                app_url = get_server_url_from_ws_url(str(websocket.url))
                blocks._queue.set_url(app_url)
            await websocket.accept()
            # In order to cancel jobs, we need the session_hash and fn_index
            # to create a unique id for each job
            try:
                await asyncio.wait_for(
                    websocket.send_json({"msg": "send_hash"}), timeout=5
                )
            except AsyncTimeOutError:
                return

            try:
                session_info = await asyncio.wait_for(
                    websocket.receive_json(), timeout=5
                )
            except AsyncTimeOutError:
                return

            event = Event(
                websocket, session_info["session_hash"], session_info["fn_index"]
            )
            # set the token into Event to allow using the same token for call_prediction
            event.token = token
            event.session_hash = session_info["session_hash"]

            # Continuous events are not put in the queue  so that they do not
            # occupy the queue's resource as they are expected to run forever
            if blocks.dependencies[event.fn_index].get("every", 0):
                await cancel_tasks({f"{event.session_hash}_{event.fn_index}"})
                await blocks._queue.reset_iterators(event.session_hash, event.fn_index)
                task = run_coro_in_background(
                    blocks._queue.process_events, [event], False
                )
                set_task_name(task, event.session_hash, event.fn_index, batch=False)
            else:
                rank = blocks._queue.push(event)

                if rank is None:
                    await blocks._queue.send_message(event, {"msg": "queue_full"})
                    await event.disconnect()
                    return
                estimation = blocks._queue.get_estimation()
                await blocks._queue.send_estimation(event, estimation, rank)
            while True:
                await asyncio.sleep(1)
                if websocket.application_state == WebSocketState.DISCONNECTED:
                    return

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


Промежуточные итоги расследования
=================================

1. Приложение загружается и сидит в памяти - предоставляет api
2. app.tokens[token] - в этом массиве хранятся залогиненные пользователи
3. opts - глобальная переменная в нее при старте приложения загружаются пути из файла config.json
4. При рендере - задача ставится в очередь через вебсокеты /queue/join
5. Для получения текущего пользователя нужен экземпляр приложения - app и токен авторизации - token = app.tokens.get(token)
6. Задача на рендеринг создается так:: task = run_coro_in_background(blocks._queue.process_events, [event], False)
7. event содержит токен авторизации event.token

Наброски решения
~~~~~~~~~~~~~~~~
написать функции - в своем модуле user_config.py
get_outdir_*(app, token)
заменить вызовы
opts.outdir_*
на user_config.get_outdir_*(app, token)

проблема - где взять app и token
и красио это будет сделать тут - namegen = FilenameGenerator(p, seed, prompt, image)
проблема - где взять app и token ???
пока не знаю


class FilenameGenerator:
    def get_vae_filename(self): #get the name of the VAE file.
        if sd_vae.loaded_vae_file is None:
            return "NoneType"
        file_name = os.path.basename(sd_vae.loaded_vae_file)
        split_file_name = file_name.split('.')
        if len(split_file_name) > 1 and split_file_name[0] == '':
            return split_file_name[1] # if the first character of the filename is "." then [1] is obtained.
        else:
            return split_file_name[0]

    replacements = {
        'seed': lambda self: self.seed if self.seed is not None else '',
        'seed_first': lambda self: self.seed if self.p.batch_size == 1 else self.p.all_seeds[0],
        'seed_last': lambda self: NOTHING_AND_SKIP_PREVIOUS_TEXT if self.p.batch_size == 1 else self.p.all_seeds[-1],
        'steps': lambda self:  self.p and self.p.steps,
        'cfg': lambda self: self.p and self.p.cfg_scale,
        'width': lambda self: self.image.width,
        'height': lambda self: self.image.height,
        'styles': lambda self: self.p and sanitize_filename_part(", ".join([style for style in self.p.styles if not style == "None"]) or "None", replace_spaces=False),
        'sampler': lambda self: self.p and sanitize_filename_part(self.p.sampler_name, replace_spaces=False),
        'model_hash': lambda self: getattr(self.p, "sd_model_hash", shared.sd_model.sd_model_hash),
        'model_name': lambda self: sanitize_filename_part(shared.sd_model.sd_checkpoint_info.model_name, replace_spaces=False),
        'date': lambda self: datetime.datetime.now().strftime('%Y-%m-%d'),
        'datetime': lambda self, *args: self.datetime(*args),  # accepts formats: [datetime], [datetime<Format>], [datetime<Format><Time Zone>]
        'job_timestamp': lambda self: getattr(self.p, "job_timestamp", shared.state.job_timestamp),
        'prompt_hash': lambda self: hashlib.sha256(self.prompt.encode()).hexdigest()[0:8],
        'prompt': lambda self: sanitize_filename_part(self.prompt),
        'prompt_no_styles': lambda self: self.prompt_no_style(),
        'prompt_spaces': lambda self: sanitize_filename_part(self.prompt, replace_spaces=False),
        'prompt_words': lambda self: self.prompt_words(),
        'batch_number': lambda self: NOTHING_AND_SKIP_PREVIOUS_TEXT if self.p.batch_size == 1 or self.zip else self.p.batch_index + 1,
        'batch_size': lambda self: self.p.batch_size,
        'generation_number': lambda self: NOTHING_AND_SKIP_PREVIOUS_TEXT if (self.p.n_iter == 1 and self.p.batch_size == 1) or self.zip else self.p.iteration * self.p.batch_size + self.p.batch_index + 1,
        'hasprompt': lambda self, *args: self.hasprompt(*args),  # accepts formats:[hasprompt<prompt1|default><prompt2>..]
        'clip_skip': lambda self: opts.data["CLIP_stop_at_last_layers"],
        'denoising': lambda self: self.p.denoising_strength if self.p and self.p.denoising_strength else NOTHING_AND_SKIP_PREVIOUS_TEXT,
        'vae_filename': lambda self: self.get_vae_filename(),

    }



def save_image(image, path, basename, seed=None, prompt=None, extension='png', info=None, short_filename=False, no_prompt=False, grid=False, pnginfo_section_name='parameters', p=None, existing_info=None, forced_filename=None, suffix="", save_to_dirs=None):
    """Save an image.

    Args:
        image (`PIL.Image`):
            The image to be saved.
        path (`str`):
            The directory to save the image. Note, the option `save_to_dirs` will make the image to be saved into a sub directory.
        basename (`str`):
            The base filename which will be applied to `filename pattern`.
        seed, prompt, short_filename,
        extension (`str`):
            Image file extension, default is `png`.
        pngsectionname (`str`):
            Specify the name of the section which `info` will be saved in.
        info (`str` or `PngImagePlugin.iTXt`):
            PNG info chunks.
        existing_info (`dict`):
            Additional PNG info. `existing_info == {pngsectionname: info, ...}`
        no_prompt:
            TODO I don't know its meaning.
        p (`StableDiffusionProcessing`)
        forced_filename (`str`):
            If specified, `basename` and filename pattern will be ignored.
        save_to_dirs (bool):
            If true, the image will be saved into a subdirectory of `path`.

    Returns: (fullfn, txt_fullfn)
        fullfn (`str`):
            The full path of the saved imaged.
        txt_fullfn (`str` or None):
            If a text file is saved for this image, this will be its full path. Otherwise None.
    """
    namegen = FilenameGenerator(p, seed, prompt, image)