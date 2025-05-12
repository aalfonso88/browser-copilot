FROM python:3.11

RUN pip install poetry

WORKDIR /usr/src/app

RUN apt-get update && apt-get install -y vim less libreadline-dev && apt-get clean

COPY pyproject.toml poetry.lock README.md ./

COPY gpt_agent/ ./gpt_agent
COPY entrypoint.sh entrypoint.sh

COPY .env .env

RUN poetry install

ADD https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh wait-for-it.sh
RUN chmod +x wait-for-it.sh

ENTRYPOINT [ "./entrypoint.sh" ]

#CMD ["poetry", "run", "python", "-m", "gpt_agent"]
CMD ["poetry", "run", "uvicorn", "gpt_agent.api:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

