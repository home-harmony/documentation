Run 

    git fetch origin && git checkout main && git reset --hard origin/main

then run 

    sam validate --lint on all 3 templates (template.yaml, data-stream-template.yaml, services-template.yaml) 
    
and 

    cargo lambda build --release --arm64 for migrate_runner, cognito_pre_signup, and family_service.