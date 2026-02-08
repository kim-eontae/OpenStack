# Keystone MFA(TOTP) 활성화 절차

Keystone MFA(TOTP) 활성화 절차
        
        **전제 조건**
        
        - Horizon에서 **관리자 계정**으로 MFA를 적용할 사용자 생성
        - 해당 사용자에게 **프로젝트 + Role** 권한 부여
        
        `sudo vi /etc/keystone/keystone.conf`
        
        ```bash
        [auth]
        methods = password,token,totp,application_credential
        ```
        
        **사용자에 MFA 활성화**
        
        ```bash
        # MFA 기능 활성화
        openstack user set <USER_ID_or_NAME> --enable-multi-factor-auth
        ```
        
        **시크릿 키 생성**
        
        ```bash
        # 16자 시크릿(임의) → Base32 변환 후 패딩 제거
        head -c 20 /dev/urandom | base32 | tr -d "="
        ```
        
        **Keystone Credential에 등록**
        
        ```bash
        openstack credential create --type totp <user name> <시크릿 Key>
        ```
        
        **OTP 등록 (Google Authenticator)**
        
        ```bash
        # QR 코드 URI 생성
        export QR_URI="otpauth://totp/<USERNAME>@OpenStack?secret=<시크릿 Key>&issuer=<표시 이름>"
        ```
        
        **패키지 설치**
        
        ```bash
        sudo apt install qrencode
        ```
        
        **QR확인**
        
        ```bash
        # QR 코드 터미널 출력
        echo -n "$QR_URI" | qrencode -t ANSIUTF8
        ```
        
        **MFA 규칙 최종 적용**
        
        웹(Horizon)은 password+totp로, CLI/자동화는 application_credential로 통과시키기 위해 **두 규칙을 모두 추가**합니다. (**둘 중 하나만 만족해도 인증 성공**)
        
        ```bash
        USER="<user name>"
        USER_ID=$(openstack user show "$USER" -f value -c id)
        
        openstack user set \
          --enable-multi-factor-auth \
          --multi-factor-auth-rule password,totp \
          --multi-factor-auth-rule application_credential \
          "$USER_ID"
        ```
        
        ### 적용 확인
        
        ```bash
        openstack user show "$USER" -f json | jq '.options'
        # "multi_factor_auth_enabled": true
        # "multi_factor_auth_rules": [["password","totp"],["application_credential"]]
        ```
        
        **Horizon Web 코드 수정**
        
        1. `sudo vi /var/www/horizon/openstack_dashboard/urls.py`
        
        ```bash
        from django.urls import include, path
        
        urlpatterns = [
            re_path(r'^$', views.splash, name='splash'),
            re_path(r'^api/', include(rest.urls)),
            re_path(r'^header/', views.ExtensibleHeaderView.as_view()),
            re_path(r'', horizon.base._wrapped_include(horizon.urls)),
            path('auth/', include('openstack_auth.urls')),
        ]
        ```
        
        1. `sudo vi /var/www/horizon/openstack_dashboard/local/local_settings.py` 
        
        ```bash
        OPENSTACK_KEYSTONE_MFA_TOTP_ENABLED = True
        OPENSTACK_KEYSTONE_MFA_SUPPORT = True
        OPENSTACK_KEYSTONE_MFA_ENABLED = True
        
        AUTHENTICATION_PLUGINS = [
            'openstack_auth.plugin.password.PasswordPlugin',
            'openstack_auth.plugin.totp.TotpPlugin',
            'openstack_auth.plugin.token.TokenPlugin',
        ]
        # SSO 비활성화(테스트용)
        WEBSSO_ENABLED = False
        
        # HTTP 환경이라면 아래 두 개 False
        SESSION_COOKIE_SECURE = False
        CSRF_COOKIE_SECURE = False
        ```
        
        1. 재시작
        
        ```bash
        sudo systemctl restart apache2
        ```
        
        ### **유저 생성시 사용자 MFA 활성 자동화**
        
        **`sudo vi  /var/www/horizon/openstack_dashboard/local/local_settings.py`**
        
        ```bash
        ENABLE_DEFAULT_MFA = True
        DEFAULT_MFA_RULES = [
            ["password", "totp"],
            ["application_credential"]
        ]
        ```
        
        **`sudo vi /var/www/horizon/openstack_dashboard/dashboards/identity/users/forms.py`**
        
        ```bash
        new_user = \
            api.keystone.user_create(request,
                                     name=data['name'],
                                     email=data['email'],
                                     description=desc or None,
                                     password=data['password'],
                                     project=data['project'] or None,
                                     enabled=data['enabled'],
                                     domain=domain.id,
                                     **kwargs)
        messages.success(request,
                         _('User "%s" was successfully created.')
                         % data['name'])
        
        # ----- 여기부터 추가 -----
        try:
            if getattr(settings, "ENABLE_DEFAULT_MFA", False):
                rules = getattr(settings, "DEFAULT_MFA_RULES", [["password", "totp"], ["application_credential"]])
                kc = api.keystone.keystoneclient(request)
                kc.users.update(
                    new_user.id,
                    options={
                        "multi_factor_auth_enabled": True,
                        "multi_factor_auth_rules": rules,
                    },
                )
        except Exception as exc:
            LOG.exception("Auto-MFA setup failed for user %s: %s", new_user.id, exc)
        # ----- 여기까지 추가 -----
        ```
        
        서비스 재시작
        
        ```bash
        sudo systemctl restart apache2 || sudo systemctl restart httpd
        ```
        
        ### TOTP 추가 후 새로운 CLI 로그인 방식
        
        totp 포함된 계정 생성
        
        ```bash
        export OS_AUTH_URL=http://mncsvrt06:5000/v3
        export OS_AUTH_TYPE=v3applicationcredential
        export OS_APPLICATION_CREDENTIAL_ID=<CRED_ID>
        export OS_APPLICATION_CREDENTIAL_SECRET=<CRED_ID_SECRET>
        ```
        
        ### **Horizon TOTP QR 페이지 생성**
        
        1. **최초 로그인 시**: TOTP Credential이 없으면 QR 등록(Enroll) 페이지로 이동
        2. **이미 등록된 경우**: QR 등록 건너뛰고 바로 TOTP 코드 입력 페이지로 이동
        
        패키지 설치
        
        ```bash
        sudo apt-get install -y python3-qrcode python3-pil || sudo pip3 install qrcode[pil]
        ```
        
        **설정 추가 (admin 측에서 credential을 생성하기 위한 권한)**
        
        1.  Keystone에서 **애플리케이션 크레덴셜** 생성(관리자 계정으로)
        
        ```bash
        openstack application credential create horizon_mfa_enroll \
          --role admin --unrestricted \
          -f json | jq -r '.id + " " + .secret'
          
        # 출력된 <APP_CRED_ID> <APP_CRED_SECRET> 을 메모
        ```
        
        1.  Horizon 설정
        
        **`sudo vi /var/www/horizon/openstack_dashboard/local/local_settings.py`**
        
        ```bash
        # --- Self-enroll (first login QR) ---
        MFA_SELF_ENROLL_ENABLED = True
        MFA_SELF_ENROLL_ISSUER  = "GenCloud"      # OTP 앱에 표시될 발급자
        MFA_SELF_ENROLL_DIGITS  = 6
        MFA_SELF_ENROLL_PERIOD  = 30
        
        # Keystone 연결 (이미 OPENSTACK_KEYSTONE_URL은 설정되어 있어야 함)
        # Application Credential for server-side Keystone calls
        MFA_KS_APP_CRED_ID     = "APP_CRED_ID"
        MFA_KS_APP_CRED_SECRET = "APP_CRED_SECRET"
        MFA_KS_APP_DOMAIN      = "Default"  # admin의 도메인명
        ```
        
        1. **시크릿 생성·QR 이미지 만들기·Keystone 호출**
        
        **`sudo vi /var/www/horizon/openstack_auth/plugin/self_enroll.py`**
        
        ```bash
        import base64, secrets
        from io import BytesIO
        
        import qrcode
        import requests
        from django.conf import settings
        def _resolve_domain_id(domain_hint: str, token: str) -> str:
            """
            domain_hint가 'default'나 실제 UUID든 둘 다 처리.
            """
            if not domain_hint:
                return "default"
            # UUID 같이 보이면 그대로 사용
            if len(domain_hint) in (32, 36) and '-' in domain_hint or len(domain_hint) == 32:
                return domain_hint
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(f"{base}/domains?name={domain_hint}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            domains = r.json().get("domains", [])
            if domains:
                return domains[0]["id"]
            # 못 찾으면 Keystone 기본 도메인
            return "default"
        
        def _b32_secret(n_bytes=20):
            raw = secrets.token_bytes(n_bytes)
            b32 = base64.b32encode(raw).decode('ascii').rstrip('=')
            return b32
        
        def _qr_png_b64(uri: str) -> str:
            img = qrcode.make(uri)
            buf = BytesIO()
            img.save(buf, format='PNG')
            return base64.b64encode(buf.getvalue()).decode('ascii')
        
        def _system_token():
            url = settings.OPENSTACK_KEYSTONE_URL.rstrip('/') + '/auth/tokens'
            payload = {
                "auth": {
                    "identity": {
                        "methods": ["application_credential"],
                        "application_credential": {
                            "id": settings.MFA_KS_APP_CRED_ID,
                            "secret": settings.MFA_KS_APP_CRED_SECRET
                        }
                    }
                    # scope 제거: 앱 크레덴셜이 가진 스코프가 자동 적용됨
                }
            }
            r = requests.post(url, json=payload,
                              headers={'Content-Type': 'application/json'}, timeout=10)
            r.raise_for_status()
            return r.headers['X-Subject-Token']
        
        def _get_user_id_by_name(user_name: str, domain_hint: str, token: str) -> str:
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            # 먼저 domain_hint를 id로 확정
            dom_id = _resolve_domain_id(domain_hint, token)
            # 우선: 도메인 ID + 사용자 이름
            r = requests.get(f"{base}/users?name={user_name}&domain_id={dom_id}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            users = r.json().get("users", [])
            if users:
                return users[0]["id"]
            # fallback: 도메인 필터 없이 이름만
            r = requests.get(f"{base}/users?name={user_name}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            users = r.json().get("users", [])
            return users[0]["id"] if users else None
        
        def ensure_user_totp_and_get_qr(user_name:str, domain_id:str):
            """
            TOTP credential이 없으면 생성하고 QR data-uri(및 otpauth URI)를 리턴.
            이미 있으면 None 리턴 → 모달 표시 안 함.
            """
            if not getattr(settings, 'MFA_SELF_ENROLL_ENABLED', False):
                return None
        
            token = _system_token()
        
            # 1) username → user_id
            user_id = _get_user_id_by_name(user_name, domain_id or "default", token)
            if not user_id:
                return None
        
            # 2) credential 존재 여부 확인
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(f"{base}/credentials?type=totp&user_id={user_id}",
                             headers={'X-Auth-Token': token}, timeout=10)
            r.raise_for_status()
            if r.json().get('credentials'):
                return None  # 이미 등록되어 있으면 끝
        
            # 3) 시크릿 생성 + credential 생성
            secret = _b32_secret()
            create = {"credential": {"blob": secret, "type": "totp", "user_id": user_id}}
            r = requests.post(f"{base}/credentials", json=create,
                              headers={'X-Auth-Token': token, 'Content-Type':'application/json'},
                              timeout=10)
            r.raise_for_status()
        
            # 4) otpauth URI & QR data-uri 생성
            issuer = getattr(settings, 'MFA_SELF_ENROLL_ISSUER', 'GenCloud')
            digits = getattr(settings, 'MFA_SELF_ENROLL_DIGITS', 6)
            period = getattr(settings, 'MFA_SELF_ENROLL_PERIOD', 30)
            uri = f"otpauth://totp/{user_name}@{issuer}?secret={secret}&issuer={issuer}&digits={digits}&period={period}"
            png_b64 = _qr_png_b64(uri)
            return f"data:image/png;base64,{png_b64}", uri
        
        def has_user_totp(user_name: str, domain_id: str) -> bool:
            """해당 사용자가 이미 TOTP credential을 갖고 있는지 여부만 확인."""
            token = _system_token()
            user_id = _get_user_id_by_name(user_name, domain_id or "default", token)
            if not user_id:
                return False
            base = settings.OPENSTACK_KEYSTONE_URL.rstrip('/')
            r = requests.get(
                f"{base}/credentials?type=totp&user_id={user_id}",
                headers={'X-Auth-Token': token},
                timeout=10,
            )
            r.raise_for_status()
            return bool(r.json().get('credentials'))
        ```
        
        **로그인 분기**
        
        **`sudo vi /var/www/horizon/openstack_auth/views.py`**
        
        ```bash
        # Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        #
        #    http://www.apache.org/licenses/LICENSE-2.0
        #
        # Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
        # implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.
        import datetime
        import functools
        import logging
        
        from openstack_auth.plugin.self_enroll import (
            ensure_user_totp_and_get_qr,
            has_user_totp,
        )
        
        from django.conf import settings
        from django.contrib import auth
        from django.contrib.auth.decorators import login_required
        from django.contrib.auth import views as django_auth_views
        from django.contrib import messages
        from django import http as django_http
        from django.middleware import csrf
        from django import shortcuts
        from django.urls import reverse
        from django.utils import http
        from django.utils.translation import gettext_lazy as _
        from django.views.decorators.cache import never_cache
        from django.views.decorators.csrf import csrf_exempt
        from django.views.decorators.csrf import csrf_protect
        from django.views.decorators.debug import sensitive_post_parameters
        from django.views.generic import edit as edit_views
        from keystoneauth1 import exceptions as keystone_exceptions
        
        from openstack_auth import exceptions
        from openstack_auth import forms
        from openstack_auth import plugin
        from openstack_auth import user as auth_user
        from openstack_auth import utils
        
        LOG = logging.getLogger(__name__)
        
        CSRF_REASONS = [
            csrf.REASON_BAD_ORIGIN,
            csrf.REASON_NO_REFERER,
            csrf.REASON_BAD_REFERER,
            csrf.REASON_NO_CSRF_COOKIE,
            csrf.REASON_CSRF_TOKEN_MISSING,
            csrf.REASON_MALFORMED_REFERER,
            csrf.REASON_INSECURE_REFERER,
        ]
        
        def get_csrf_reason(reason):
            if not reason:
                return
        
            if reason not in CSRF_REASONS:
                reason = ""
            else:
                reason += " "
            reason += str(_("Cookies may be turned off. "
                            "Make sure cookies are enabled and try again."))
            return reason
        
        def set_logout_reason(res, msg):
            msg = msg.encode('unicode_escape').decode('ascii')
            res.set_cookie('logout_reason', msg, max_age=10)
        
        def is_ajax(request):
            # See horizon.utils.http.is_ajax() for more detail.
            # NOTE: openstack_auth does not import modules from horizon to avoid
            # import loops, so we copy the logic from horizon.utils.http.
            return request.headers.get('x-requested-with') == 'XMLHttpRequest'
        
        # TODO(stephenfin): Migrate to CBV
        @sensitive_post_parameters()
        @csrf_protect
        @never_cache
        def login(request):
            """Logs a user in using the :class:`~openstack_auth.forms.Login` form."""
        
            # If the user enabled websso and the default redirect
            # redirect to the default websso url
            if (request.method == 'GET' and settings.WEBSSO_ENABLED and
                    settings.WEBSSO_DEFAULT_REDIRECT):
                protocol = settings.WEBSSO_DEFAULT_REDIRECT_PROTOCOL
                region = settings.WEBSSO_DEFAULT_REDIRECT_REGION
                origin = utils.build_absolute_uri(request, '/auth/websso/')
                url = ('%s/auth/OS-FEDERATION/websso/%s?origin=%s' %
                       (region, protocol, origin))
                return shortcuts.redirect(url)
        
            # If the user enabled websso and selects default protocol
            # from the dropdown, We need to redirect user to the websso url
            if request.method == 'POST':
                auth_type = request.POST.get('auth_type', 'credentials')
                request.session['auth_type'] = auth_type
                if settings.WEBSSO_ENABLED and auth_type != 'credentials':
                    region_id = request.POST.get('region')
                    auth_url = getattr(settings, 'WEBSSO_KEYSTONE_URL', None)
                    if auth_url is None:
                        auth_url = forms.get_region_endpoint(region_id)
                    url = utils.get_websso_url(request, auth_url, auth_type)
                    return shortcuts.redirect(url)
        
            if not is_ajax(request):
                # If the user is already authenticated, redirect them to the
                # dashboard straight away, unless the 'next' parameter is set as it
                # usually indicates requesting access to a page that requires different
                # permissions.
                if (request.user.is_authenticated and
                        auth.REDIRECT_FIELD_NAME not in request.GET and
                        auth.REDIRECT_FIELD_NAME not in request.POST):
                    return shortcuts.redirect(settings.LOGIN_REDIRECT_URL)
        
            # Get our initial region for the form.
            initial = {}
            current_region = request.session.get('region_endpoint', None)
            requested_region = request.GET.get('region', None)
            regions = dict(settings.AVAILABLE_REGIONS)
            if requested_region in regions and requested_region != current_region:
                initial.update({'region': requested_region})
        
            if request.method == "POST":
                form = functools.partial(forms.Login)
            else:
                form = functools.partial(forms.Login, initial=initial)
        
            choices = settings.WEBSSO_CHOICES
            reason = get_csrf_reason(request.GET.get('csrf_failure'))
            logout_reason = request.COOKIES.get(
                'logout_reason', '').encode('ascii').decode('unicode_escape')
            logout_status = request.COOKIES.get('logout_status')
            extra_context = {
                'redirect_field_name': auth.REDIRECT_FIELD_NAME,
                'csrf_failure': reason,
                'show_sso_opts': settings.WEBSSO_ENABLED and len(choices) > 1,
                'classes': {
                    'value': '',
                    'single_value': '',
                    'label': '',
                },
                'logout_reason': logout_reason,
                'logout_status': logout_status,
            }
        
            if is_ajax(request):
                template_name = 'auth/_login.html'
                extra_context['hide'] = True
            else:
                template_name = 'auth/login.html'
        
            try:
                res = django_auth_views.LoginView.as_view(
                    template_name=template_name,
                    redirect_field_name=auth.REDIRECT_FIELD_NAME,
                    form_class=form,
                    extra_context=extra_context,
                    redirect_authenticated_user=False)(request)
            
            except exceptions.KeystoneTOTPRequired as exc:
                username = request.POST.get('username')
                domain   = request.POST.get('domain')
                # 이미 세션에 receipt, domain 저장
                request.session['receipt'] = exc.receipt
                request.session['domain']  = domain
        
                # self-enroll이 켜져 있고, 사용자에게 아직 TOTP가 없으면 enroll로,
                # 있으면 바로 TOTP 입력 페이지로.
                try:
                    if getattr(settings, "MFA_SELF_ENROLL_ENABLED", False) and not has_user_totp(username, domain):
                        res = django_http.HttpResponseRedirect(
                            reverse('totp-enroll', args=[username])
                        )
                    else:
                        res = django_http.HttpResponseRedirect(
                            reverse('totp', args=[username])
                        )
                except Exception:
                    # 체크 실패 시 안전하게 TOTP 입력으로 보냄
                    res = django_http.HttpResponseRedirect(
                        reverse('totp', args=[username])
                    )
        
            except exceptions.KeystonePassExpiredException as exc:
                res = django_http.HttpResponseRedirect(
                    reverse('password', args=[exc.user_id]))
                msg = _("Your password has expired. Please set a new password.")
                set_logout_reason(res, msg)
        
            # Save the region in the cookie, this is used as the default
            # selected region next time the Login form loads.
            if request.method == "POST":
                utils.set_response_cookie(res, 'login_region',
                                          request.POST.get('region', ''))
                utils.set_response_cookie(res, 'login_domain',
                                          request.POST.get('domain', ''))
        
            # Set the session data here because django's session key rotation
            # will erase it if we set it earlier.
            if request.user.is_authenticated:
                auth_user.set_session_from_user(request, request.user)
                regions = dict(forms.get_region_choices())
                region = request.user.endpoint
                login_region = request.POST.get('region')
                region_name = regions.get(login_region)
                request.session['region_endpoint'] = region
                request.session['region_name'] = region_name
                # Check for a services_region cookie. Fall back to the login_region.
                services_region = request.COOKIES.get('services_region', region_name)
                if services_region in request.user.available_services_regions:
                    request.session['services_region'] = services_region
                expiration_time = request.user.time_until_expiration()
                threshold_days = settings.PASSWORD_EXPIRES_WARNING_THRESHOLD_DAYS
                if (expiration_time is not None and
                        expiration_time.days <= threshold_days and
                        expiration_time > datetime.timedelta(0)):
                    expiration_time = str(expiration_time).rsplit(':', 1)[0]
                    msg = (_('Please consider changing your password, it will expire'
                             ' in %s minutes') %
                           expiration_time).replace(':', ' Hours and ')
                    messages.warning(request, msg)
            return res
        
        # TODO(stephenfin): Migrate to CBV
        @sensitive_post_parameters()
        @csrf_exempt
        @never_cache
        def websso(request):
            """Logs a user in using a token from Keystone's POST."""
            if settings.WEBSSO_USE_HTTP_REFERER:
                referer = request.META.get('HTTP_REFERER',
                                           settings.OPENSTACK_KEYSTONE_URL)
                auth_url = utils.clean_up_auth_url(referer)
            else:
                auth_url = settings.OPENSTACK_KEYSTONE_URL
            token = request.POST.get('token')
            try:
                request.user = auth.authenticate(request, auth_url=auth_url,
                                                 token=token)
            except exceptions.KeystoneAuthException as exc:
                if settings.WEBSSO_DEFAULT_REDIRECT:
                    res = django_http.HttpResponseRedirect(settings.LOGIN_ERROR)
                else:
                    msg = 'Login failed: %s' % exc
                    res = django_http.HttpResponseRedirect(settings.LOGIN_URL)
                    set_logout_reason(res, msg)
                return res
        
            auth_user.set_session_from_user(request, request.user)
            auth.login(request, request.user)
            if request.session.test_cookie_worked():
                request.session.delete_test_cookie()
            return django_http.HttpResponseRedirect(settings.LOGIN_REDIRECT_URL)
        
        # TODO(stephenfin): Migrate to CBV
        def logout(request, login_url=None, **kwargs):
            """Logs out the user if he is logged in. Then redirects to the log-in page.
        
            :param login_url:
                Once logged out, defines the URL where to redirect after login
        
            :param kwargs:
                see django.contrib.auth.views.logout_then_login extra parameters.
        
            """
            msg = 'Logging out user "%(username)s".' % \
                {'username': request.user.username}
            LOG.info(msg)
        
            """ Securely logs a user out. """
            if (settings.WEBSSO_ENABLED and settings.WEBSSO_DEFAULT_REDIRECT and
                    settings.WEBSSO_DEFAULT_REDIRECT_LOGOUT):
                auth_user.unset_session_user_variables(request)
                return django_http.HttpResponseRedirect(
                    settings.WEBSSO_DEFAULT_REDIRECT_LOGOUT)
        
            return django_auth_views.logout_then_login(request,
                                                       login_url=login_url,
                                                       **kwargs)
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch(request, tenant_id, redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches an authenticated user from one project to another."""
            LOG.debug('Switching to tenant %s for user "%s".',
                      tenant_id, request.user.username)
        
            endpoint, __ = utils.fix_auth_url_version_prefix(request.user.endpoint)
            client_ip = utils.get_client_ip(request)
            session = utils.get_session(original_ip=client_ip)
            # Keystone can be configured to prevent exchanging a scoped token for
            # another token. Always use the unscoped token for requesting a
            # scoped token.
            unscoped_token = request.user.unscoped_token
            auth = utils.get_token_auth_plugin(auth_url=endpoint,
                                               token=unscoped_token,
                                               project_id=tenant_id)
        
            try:
                auth_ref = auth.get_access(session)
                msg = 'Project switch successful for user "%(username)s".' % \
                    {'username': request.user.username}
                LOG.info(msg)
            except keystone_exceptions.ClientException:
                msg = (
                    _('Project switch failed for user "%(username)s".') %
                    {'username': request.user.username})
                messages.error(request, msg)
                auth_ref = None
                LOG.exception('An error occurred while switching sessions.')
        
            # Ensure the user-originating redirection url is safe.
            # Taken from django.contrib.auth.views.login()
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            if auth_ref:
                user = auth_user.create_user_from_token(
                    request,
                    auth_user.Token(auth_ref, unscoped_token=unscoped_token),
                    endpoint)
                auth_user.set_session_from_user(request, user)
                message = (
                    _('Switch to project "%(project_name)s" successful.') %
                    {'project_name': request.user.project_name})
                messages.success(request, message)
            response = shortcuts.redirect(redirect_to)
            utils.set_response_cookie(response, 'recent_project',
                                      request.user.project_id)
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_region(request, region_name,
                          redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches the user's region for all services except Identity service.
        
            The region will be switched if the given region is one of the regions
            available for the scoped project. Otherwise the region is not switched.
            """
            if region_name in request.user.available_services_regions:
                request.session['services_region'] = region_name
                LOG.debug('Switching services region to %s for user "%s".',
                          region_name, request.user.username)
        
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            response = shortcuts.redirect(redirect_to)
            utils.set_response_cookie(response, 'services_region',
                                      request.session['services_region'])
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_keystone_provider(request, keystone_provider=None,
                                     redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches the user's keystone provider using K2K Federation
        
            If keystone_provider is given then we switch the user to
            the keystone provider using K2K federation. Otherwise if keystone_provider
            is None then we switch the user back to the Identity Provider Keystone
            which a non federated token auth will be used.
            """
            base_token = request.session.get('k2k_base_unscoped_token', None)
            k2k_auth_url = request.session.get('k2k_auth_url', None)
            keystone_providers = request.session.get('keystone_providers', None)
            recent_project = request.COOKIES.get('recent_project')
        
            if not base_token or not k2k_auth_url:
                msg = _('K2K Federation not setup for this session')
                raise exceptions.KeystoneAuthException(msg)
        
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            unscoped_auth_ref = None
            keystone_idp_id = settings.KEYSTONE_PROVIDER_IDP_ID
        
            if keystone_provider == keystone_idp_id:
                current_plugin = plugin.TokenPlugin()
                unscoped_auth = current_plugin.get_plugin(auth_url=k2k_auth_url,
                                                          token=base_token)
            else:
                # Switch to service provider using K2K federation
                plugins = [plugin.TokenPlugin()]
                current_plugin = plugin.K2KAuthPlugin()
        
                unscoped_auth = current_plugin.get_plugin(
                    auth_url=k2k_auth_url, service_provider=keystone_provider,
                    plugins=plugins, token=base_token, recent_project=recent_project)
        
            try:
                # Switch to identity provider using token auth
                unscoped_auth_ref = current_plugin.get_access_info(unscoped_auth)
            except exceptions.KeystoneAuthException as exc:
                msg = 'Switching to Keystone Provider %s has failed. %s' \
                      % (keystone_provider, exc)
                messages.error(request, msg)
        
            if unscoped_auth_ref:
                try:
                    request.user = auth.authenticate(
                        request, auth_url=unscoped_auth.auth_url,
                        token=unscoped_auth_ref.auth_token)
                except exceptions.KeystoneAuthException as exc:
                    msg = 'Keystone provider switch failed: %s' % exc
                    res = django_http.HttpResponseRedirect(settings.LOGIN_URL)
                    set_logout_reason(res, msg)
                    return res
                auth.login(request, request.user)
                auth_user.set_session_from_user(request, request.user)
                request.session['keystone_provider_id'] = keystone_provider
                request.session['keystone_providers'] = keystone_providers
                request.session['k2k_base_unscoped_token'] = base_token
                request.session['k2k_auth_url'] = k2k_auth_url
                message = (
                    _('Switch to Keystone Provider "%(keystone_provider)s" '
                      'successful.') % {'keystone_provider': keystone_provider})
                messages.success(request, message)
        
            response = shortcuts.redirect(redirect_to)
            return response
        
        # TODO(stephenfin): Migrate to CBV
        @login_required
        def switch_system_scope(request, redirect_field_name=auth.REDIRECT_FIELD_NAME):
            """Switches an authenticated user from one system to another."""
            LOG.debug('Switching to system scope for user "%s".', request.user.username)
        
            endpoint, __ = utils.fix_auth_url_version_prefix(request.user.endpoint)
            client_ip = utils.get_client_ip(request)
            session = utils.get_session(original_ip=client_ip)
            # Keystone can be configured to prevent exchanging a scoped token for
            # another token. Always use the unscoped token for requesting a
            # scoped token.
            unscoped_token = request.user.unscoped_token
            auth = utils.get_token_auth_plugin(auth_url=endpoint,
                                               token=unscoped_token,
                                               system_scope='all')
        
            try:
                auth_ref = auth.get_access(session)
            except keystone_exceptions.ClientException:
                msg = (
                    _('System switch failed for user "%(username)s".') %
                    {'username': request.user.username})
                messages.error(request, msg)
                auth_ref = None
                LOG.exception('An error occurred while switching sessions.')
            else:
                msg = 'System switch successful for user "%(username)s".' % \
                    {'username': request.user.username}
                LOG.info(msg)
        
            # Ensure the user-originating redirection url is safe.
            # Taken from django.contrib.auth.views.login()
            redirect_to = request.GET.get(redirect_field_name, '')
            if (not http.url_has_allowed_host_and_scheme(
                    url=redirect_to,
                    allowed_hosts=[request.get_host()])):
                redirect_to = settings.LOGIN_REDIRECT_URL
        
            if auth_ref:
                user = auth_user.create_user_from_token(
                    request,
                    auth_user.Token(auth_ref, unscoped_token=unscoped_token),
                    endpoint)
                auth_user.set_session_from_user(request, user)
                message = _('Switch to system scope successful.')
                messages.success(request, message)
            response = shortcuts.redirect(redirect_to)
            return response
        
        class PasswordView(edit_views.FormView):
            """Changes user's password when it's expired or otherwise inaccessible."""
            template_name = 'auth/password.html'
            form_class = forms.Password
            success_url = settings.LOGIN_URL
        
            def get_initial(self):
                return {
                    'user_id': self.kwargs['user_id'],
                    'region': self.request.COOKIES.get('login_region'),
                }
        
            def form_valid(self, form):
                # We have no session here, so regular messages don't work.
                msg = _('Password changed. Please log in to continue.')
                res = django_http.HttpResponseRedirect(self.success_url)
                set_logout_reason(res, msg)
                return res
        
        class TotpView(edit_views.FormView):
            """Logs a user in using a TOTP authentification"""
            template_name = 'auth/totp.html'
            form_class = forms.TimeBasedOneTimePassword
            success_url = settings.LOGIN_REDIRECT_URL
            fail_url = "/login/"
        
            def get_initial(self):
                return {
                    'request': self.request,
                    'username': self.kwargs['user_name'],
                    'receipt': self.request.session.get('receipt'),
                    'region': self.request.COOKIES.get('login_region'),
                    'domain': self.request.session.get('domain'),
                }
            
            def form_valid(self, form):
                auth.login(self.request, form.user_cache)
                res = django_http.HttpResponseRedirect(self.success_url)
                request = self.request
                if self.request.user.is_authenticated:
                    del request.session['receipt']
                    auth_user.set_session_from_user(request, request.user)
                    regions = dict(forms.get_region_choices())
                    region = request.user.endpoint
                    login_region = request.POST.get('region')
                    region_name = regions.get(login_region)
                    request.session['region_endpoint'] = region
                    request.session['region_name'] = region_name
                return res
        
            def form_invalid(self, form):
                if 'KeystoneNoBackendException' in str(form.errors):
                    return django_http.HttpResponseRedirect(self.fail_url)
                return super().form_invalid(form)
        
        @never_cache
        def totp_enroll(request, user_name):
            """최초 1회 QR을 작은 사이즈로 보여주고, 3분 카운트다운 후 TOTP 입력 화면으로 보냄."""
            domain_id = request.session.get('domain')
            qr_data_uri = None
            otpauth_uri = None
        
            try:
                out = ensure_user_totp_and_get_qr(user_name=user_name, domain_id=domain_id)
                if out:
                    qr_data_uri, otpauth_uri = out
            except Exception as e:
                LOG.exception("MFA enroll page error for %s: %s", user_name, e)
        
            ctx = {
                'username': user_name,
                'qr_data_uri': qr_data_uri,
                'otpauth_uri': otpauth_uri,
                'countdown_seconds': 180,   # 3분
            }
            return shortcuts.render(request, 'auth/totp_enroll.html', ctx)
        ```
        
        **URL 라우팅 추가**
        
        **`sudo vi /var/www/horizon/openstack_dashboard/urls.py`** 
        
        ```bash
        from openstack_auth import views as auth_views
        
        urlpatterns = [
            # ... 기존 항목들 ...
            path('auth/totp/enroll/<str:user_name>/', auth_views.totp_enroll, name='totp-enroll'),
        ]
        ```
        
        **Enroll 템플릿(QR 화면)**
        
        폴더 생성
        
        ```bash
        sudo mkdir -p /var/www/horizon/openstack_auth/templates/auth/
        sudo chown -R www-data:www-data /var/www/horizon/openstack_auth/templates
        ```
        
        **`sudo vi /var/www/horizon/openstack_auth/templates/auth/totp_enroll.html`**
        
        ```bash
        {% extends "auth/login.html" %}
        {% load i18n %}
        
        {% block content %}
        <div class="container">
          <div class="row">
            <div class="col-sm-6 col-sm-offset-3">
              <div class="panel panel-default" style="margin-top:12px">
                <div class="panel-heading">
                  <strong>TOTP {% trans "등록 (최초 1회)" %}</strong>
                </div>
                <div class="panel-body" style="text-align:center">
                  {% if qr_data_uri %}
                    <p style="margin-bottom:8px">
                      {% trans "아래 QR을 인증 앱(예: Google Authenticator)으로 스캔하세요." %}
                    </p>
                    <img src="{{ qr_data_uri }}" alt="TOTP QR"
                         style="max-width:160px;height:auto;display:block;margin:0 auto 8px auto;">
                    {% if otpauth_uri %}
                      <div style="font-size:12px;color:#666;word-break:break-all;margin-bottom:8px">
                        {{ otpauth_uri }}
                      </div>
                    {% endif %}
                    <div style="margin-top:6px">
                      <span id="mfa-timer" style="font-weight:bold">03:00</span>
                      <span style="color:#888"> / {% trans "3분 후 자동 진행" %}</span>
                    </div>
                    <div style="margin-top:12px">
                      <a id="mfa-confirm" class="btn btn-primary">{% trans "확인" %}</a>
                    </div>
                  {% else %}
                    <p style="color:#c00">
                      {% trans "QR 생성에 실패했습니다. 잠시 후 다시 시도하세요." %}
                    </p>
                    <div style="margin-top:12px">
                      <a href="{% url 'totp' username %}" class="btn btn-default">{% trans "TOTP 입력으로 이동" %}</a>
                    </div>
                  {% endif %}
                </div>
              </div>
        
              <!-- 아래는 기본 로그인 조각을 안 보이게 하고 싶을 때(선택) -->
              <style>
                /* 원하면 기본 로그인 폼을 가릴 수 있음 */
                form[action$="/auth/totp/{{ username }}/"] { display:none; }
              </style>
            </div>
          </div>
        </div>
        
        <script>
        (function () {
          var left = {{countdown_seconds|default:180}};
          var timerEl = document.getElementById('mfa-timer');
          function pad(n){ return (n<10?'0':'')+n; }
          function goNext(){ window.location.href = "{% url 'totp' username %}"; }
          var confirmBtn = document.getElementById('mfa-confirm');
          if (confirmBtn) confirmBtn.addEventListener('click', goNext);
          function tick(){
            if (timerEl){
              var m = Math.floor(left/60), s = left%60;
              timerEl.textContent = pad(m)+":"+pad(s);
            }
            if (left <= 0) { goNext(); return; }
            left -= 1; setTimeout(tick, 1000);
          }
          tick();
        })();
        </script>
        {% endblock %}
        ```
        
    -
